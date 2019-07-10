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
- Test cases isolation, e.g. ability to run single traffic test cases without a need to run whole test
- Avoid duplications like in acltb_test_*.yml playbooks


### Framework requirements

pytest-ansible plugin does not provide all the functionality it may be required for implementing ACL test

##### 'ansible_*' fixtures scope limitation

Because of 'ansible_adhoc' fixture is scoped to a function it limits the possibility to use this fixture in class/module/session scoped setup/teardown style fixtures.

There is no obious reason why it was limited to 'function'. Unless SONiC system test framework uses 'ansible_adhoc' in a way it breaks the 'session' scope, we could override the fixture with 'session' scope in 'conftest.py'.

NOTE: *We will skip this part here in the design right now until we reach some decision on this limitation; Below fixutres in this design depend on 'dut' host provided by 'ansible_adhoc'*


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
    acstests/ -> ../ansible/roles/test/files/acstests/
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


#### test_acl.py

The current test is divided into 4 cases:
- Basic test - setup ACL rules, run traffic, teardown ACL rules
- Incremental test - setup ACL rules incrementaly, run traffic, teardown ACL rules
- Reboot test - setup ACL rules, reboot, run traffic, teardown ACL rules
- Port toggle test - setup ACL rules, toggle all ports, run traffic, teardown ACL rules

The test logic will be seperated into generic traffic test in *BaseAclTest* and derivatives are meant to provide fixture to setup/teardown according to their requirements.

According to above cases 4 test classes has to be coded:

- *TestAcl*
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
def setup(dut, testbed):
    '''
    Setup fixture gathers all test required information from DUT facts and testbed
    '''

@pytest.fixture(scope='module', params=['ingress', 'egress'])
def stage(request):
    '''
    Small fixture to return the ACL table stage is beeing tested
    '''

@pytest.yield_fixture(scope='module')
def config(request, dut, setup):
    '''
    Fixture generates ACL tables, ACL rules configuration and deploys on the DUT
    The return value is config files generated
    '''

@pytest.yield_fixture(scope='module')
def table(dut, config, stage):
    '''
    Create an ACL table on DUT
    '''

```

Based on the above it is assumed that tests in one module will reuse the ACL table created by 'table' fixture.

If, for example, some one is developing ACL table scale test he will need a seperate test module ('test_acl_scale.py') and override 'config' fixutre to generate many tables configuration and reuse 'table' fixture.

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
        Setup/Teardown ACL rules on DUT
        '''

    @pytest.yield_fixture(scope='class', autouse=True)
    def counters_sanity_check(self, dut, rules):
        '''
        Sanity checks for ACL counters: verify ACL counters increased
        '''

    def test_traffic(self):
        # ...

```


#### Isolate test cases in PTF script *acltb_test.py*

Current implementation of the PTF test includes all the test cases under signle test object. There are two issue with current approach
- Test cases are not isolated, so user cannot run individual test without modifying sources mannualy
- Test cases hardcode ACL rule behaviour, this means we have ACL rule configuration generation step which defines which rules to test and PTF test script must be aligned with configuration; we will try to have PTF test very generic for different ACL rules and py.test logic will invoke PTF with parameters aligned with generated ACL configuration

PTF test script will have a base class with defined methods to send/receive:

```python
class AclRuleTest(BaseTest):
    def setUp(self):
        # generic parameters from self.test_params
        pass

    def runTest(self):
        # run test

class L4SourcePortMatch(AclRuleTest):
    def setUp(self):
        super(L4PortRangeMatch, self).setUp()
        self.sport = self.test_params['source_port']
        self.pkt.sport = self.exp_pkt.sport = self.sport
```

Invokation like this:

```
ptf \
    --test-dir acstests acltb_test.L4SourcePortMatch
    --platform-dir ptftests
    --platform remote
    -t "
        # ...
        proto='udp',
        source_port='4126';
        direction='spine->tor';
        forward='True';
       "
```

Will run the test for L4 source port match ACL rule wich is expected to accept traffic

###### Pytest code

In py.test script the logic will look like this:

```python

class BaseAclTest(object):

    @pytest.mark.parametrize("test_case, proto, forward, direction, options",
        [
            (
                "L4SourcePortMatch",
                "tcp",
                'forward',
                "spine->tor",
                {
                    "sport": "4126",
                }
            ),
        ]
    )
    def test_traffic(self, ansible_adhoc, testbed, test_case, options):
        # invoke ptf runner
```

Ideally the parametrization list can be autogenerated based on rules configuration.

Adding a new rule and creating a test case for it assumes creating or reusing existing test class and adding the test case in the parametrized list.

Run result output:

```
[       INFO ] setup ACL rules on arc-switch1025
...
test_acl.py::TestAcl::test_traffic[ingress-SouceIpMatch-forward-spine->tor-options1] PASSED
test_acl.py::TestAcl::test_traffic[egress-DestIpMatch-drop-spine->tor-options2] FAILED
...
[       INFO ] teardown ACL rules on arc-switch1025
...
```

#### Loganalyzer

Any fixture the executes DUT commands will run with loganalyzer. If some setup stage fails with errors in logs the test cases that use those ACL rules will fail, however other tests that do not rely on failed ACL rules will still run.

#### Marks

IMO, marks are usefull

Use cases:

1. One has modified loganalyzer and wants to make sure he did not break tests, so wants to run all tests that use loganalyzer, like this:
```
py.test -m loganalyzer
```

2. Assuming, one wants to run warm reboot tests, but warm reboot can be invoked in many tests: pfc wd, vxlan, warm reboot test itself.

```
py.test -m warm_reboot
```

For ACL test the following marks are suggested:
- loganalyzer (whole module)
- reboot (test class)
- port_toggle (test class)

#### Wrapping in ansible

acltb.yml will be modified to print message:

```
The ansible playbook for ACL test is depreceted, use py.test. This playbook is just a wrapper.
```

And execute local action to invoke py.test

