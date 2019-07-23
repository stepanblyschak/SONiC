# ACL test migration to py.test HLD

### The existing test plan and scripts

The existing test plan and scripts: [ACL test plan](https://github.com/Azure/SONiC/wiki/ACL-test-plan])

#### Test structure

```
ansible/
    roles/
        test/
            tasks/
                acltb.yml
                acl/
                    acltb/
                        acltb_config.yml
                        acltb_config_generate.yml
                        acltb_run_test.yml
                        acltb_test_basic.yml
                        acltb_test_port_toggle.yml
                        acltb_test_reboot.yml
                   acltb_expect_messages.txt
                   acltb_ignore_messages.txt
                   acltb_match_messages.txt
                   acltb_test_rules.json
                   acltb_test_rules_part_1.json
                   acltb_test_rules_part_2.json
            files/
                acstests/
                    acltb_test.py
```

- *acltb.yml*

    The main entry point for the test.

- *acltb_config.yml*/*acltb_config_generate.yml*

    Setting up the test variables; ACL tables, rules configuration generation

- *acltb_run_test.yml*

    Apply generated configuration & run *acltb_test.basic.yml*, *acltb_test_port_toggle.yml*, *acltb_test_reboot.yml*

- *acltb_test_<basic|port_togle|reboot.yml*

    Actual tests

### Scope of this document

- we will cover everything that acltb.yml roles does and suggest an approach to convert it to pytest
- we will skip basic sanity checks required for all tests - this is not the scope of this document

### Requirements

- Provide the same coverage
- Test cases independency, e.g. ability to run single traffic test cases without a need to run whole test
- Care about test performance (whole ACL test cases take ~40m)
- Avoid duplications like in acltb_test_*.yml playbooks
- Previous command line to run ansible ACL test should invoke pytest

### Framework requirements

##### 'ansible_*' fixtures scope limitation

Because of 'ansible_adhoc' fixture is scoped to a function it limits the possibility to use this fixture in class/module/session scoped setup/teardown style fixtures.

There is no obious reason why it was limited to 'function'. Unless SONiC system test framework uses 'ansible_adhoc' in a way it breaks the 'session' scope, we could override the fixture with 'session' scope in 'conftest.py'.

##### duthost, ptfhost fixtures shortcuts

To avoid repeating same code snippet we will provide two shortcut fixtures *duthost* and *ptfhost* in 'conftest.py'

Simplifies this:
```python
def test_smth(ansible_adhoc, testbed):
    dut = testbed['dut']
    duthost = ansible_host(ansible_adhoc, dut)
    # ...
```
to this:
```python
def test_smth(duthost):
    # ...
```


##### Common tasks missing

The following has to be implemented in the same way as in relevant playbook roles:

- reboot
- reload config
- port toggle

### Design

#### Directory structure

The test will be structured as follows:

```
tests/
    common/
        __init__.py
        reload_config.py
        reboot.py
        port_toggle.py
    acl/
        conftest.py
        test_acl.py
        templates/
        loganalyzer/
```

- The *acl* directory here created, so that other test modules related to ACL test can be put here and ACL related fixtures, etc. can live in conftest.py
- The *common* has common functionallity required by a set of tests.
- *acstests* simply a symbolic link to acs PTF test scripts
- *templates* direcotry contains ACL related templates (configuration for ACL table, rules)
- *loganalyzer* directory contains loganalyzer match/expect/ignore files


#### test_acl.py

The current test is divided into 4 cases:
- Basic test - setup ACL rules, run traffic, teardown ACL rules
- Incremental test - setup ACL rules incrementaly, run traffic, teardown ACL rules
- Reboot test - setup ACL rules, reboot, run traffic, teardown ACL rules
- Port toggle test - setup ACL rules, toggle all ports, run traffic, teardown ACL rules

The traffic test logic will be seperated into generic traffic test in *BaseAclTest* and derivatives are meant to provide fixture to setup/teardown according to their requirements.

According to above cases 4 test classes has to be coded:

- *TestBasicAcl*
- *TestIncrementalAcl*
- *TestAclWithReboot*
- *TestAclWithPortToggle*


In common for all above cases we need to do the following setup/teardown:
- setup test variables according to topology and minigraph facts
e.g. get the list of ToR/Spine ports, setup names and configuration for ACL tables and so on
- generate and deploy configuration files to the DUT, after the test remove configuration and cleanup

Also configuration generation and dependent traffic run tests will be parametrized for 'ingress', 'egress' ACL tests

##### Test fixtures

As above described setup and configuration generation can be done in scope of *test_acl.py* module only once.

These fixtures, by the way,can live in *acl/conftest.py*, so other ACL test modules can reuse them.

###### Module scope

```python
@pytest.fixture(scope='module')
def setup(duthost, testbed):
    '''
    Setup fixture gathers all test required information from DUT facts and testbed and returns a dictionary of gathered information
    E.g.
      - router_mac
      - tor_ports
      - spine_ports
      - dest IP tor blocked
      - dest IP tor forwarded
      - dest IP spine blocked
      - dest IP spine forwarded
      - etc.
    '''

@pytest.fixture(scope='module', params=['ingress', 'egress'])
def stage(request):
    '''
    Small fixture to parametrize test for ingress, egress tables
    '''

@pytest.yield_fixture(scope='module')
def acl_table_config(request, dut, setup, stage):
    '''
    Fixture generates ACL tables configuration files before yield and removes generated files after yield
    '''

@pytest.yield_fixture(scope='module')
def acl_table(dut, acl_table_config):
    '''
    Apply ACL table on DUT before yield, remove ACL table after yield
    '''

```

Based on the above it is assumed that tests in one module will reuse the ACL table created by 'acl_table' fixture.

If, for example, some one is developing ACL table scale test he will need a seperate test module ('test_acl_scale.py') and override 'config' fixutre to generate many tables configuration and reuse 'acl_table' fixture to apply scale configuration.

###### Class scope


```python

class BaseAclTest(object):
    __meta__ = abc.ABCMeta

    @abc.abstractmethod
    def setup_rules(self):
        pass

    @abc.abstractmethod
    def teardown_rules(self):
        pass

    @pytest.yield_fixture(scope='class')
    def rules(self, dut, table):
        '''
        Call setup/teardown ACL rules on DUT
        '''
        
    @pytest.fixture(params=['spine->tor', 'tor->spine'])
    def direction(self, request):
        '''
        fixture used to parametrize traffic test cases for two different directions
        '''

    @pytest.yield_fixture(scope='class', autouse=True)
    def counters_sanity_check(self, dut, rules):
        '''
        Sanity checks for ACL counters: verify ACL counters increased.
        This fixture should yield a 'set' of rule IDs to verify counters have increased after traffic test.
        The traffic test cases have to append rule ID in the set.
        Before traffic test cases start this fixture collects counters using 'acl_facts' ansible module;
        After traffic test cases this fixture goes through the set of rule IDs and verify counters have increased
        '''
```


#### Traffic test cases

##### traffic test cases will be implemented according to proposal https://github.com/stepanblyschak/SONiC/blob/ptf_pytest/doc/pytest/ptf_pytest_integration.md

e.g. for ICMP source IP match rule with action DROP test case

```python
def test_icmp_source_ip_match_dropped(self, setup, direction, ptf_adapter, counters_sanity_check):
    pkt = self.icmp_packet(setup, direction, ptf_adapter)
    pkt['IP'].src = '20.0.0.8'
    exp_pkt = self.expected_mask_routed_packet(pkt)

    testutils.send(ptf_adapter, self.get_src_port(setup, direction), pkt)
    testutils.verify_no_packet_any(ptf_adapter, exp_pkt, ports=self.get_dst_ports(setup, direction))
```

#### test run output example

```
acl/test_acl.py::TestBasicAcl::test_rules_priority_dropped[ingress-tor->spine] PASSED                                 [  3%]
acl/test_acl.py::TestBasicAcl::test_rules_priority_dropped[ingress-spine->tor] PASSED                                 [  3%]
...
acl/test_acl.py::TestBasicAcl::test_icmp_source_ip_match_forwarded[egress-tor->spine] PASSED                          [ 76%]
acl/test_acl.py::TestBasicAcl::test_icmp_source_ip_match_forwarded[egress-spine->tor] PASSED                          [ 76%]

```

#### Loganalyzer

Any fixture the executes DUT commands will run with loganalyzer configured with *tests/acl/loganalyzer/* files. If some setup stage fails with errors in logs the test cases that use those ACL rules will fail, however other tests that do not rely on failed ACL rules will still run.

#### Marks

- mark whole test with mark 'acl'
- mark *TestAclWithReboot* test with mark 'reboot'

#### Wrapping in ansible

##### *pytest_runner.yml* will be created in *ansible/roles/test/tasks*:

This playbook will have input variables:

```yml

- include: pytest_runner.yml
  vars:
    test_node: acl # test directory or file or specific test case to run
    test_filter: '{{ test_filter_expr }}' # based on ansible input parameters some test have to be skipped, this can be done via *test_filter* which is passed to pytest as this : 'pytest acl -k '{{ test_filter }}'
    test_mark: acl # optionaly run test by mark

```

It will print a message that ansible-playbook is deprecated for this test:

```
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!! Ansible playbook for running ACL test is now deprecated !!!!
!!!! This playbook is just a wrapper to run py.test !!!!!!!!!!!!!
!!!!!!!!!!!!!! Consider running it with py.test !!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
```

acltb.yml will be modified to invoke *pytest_runner.yml*:

