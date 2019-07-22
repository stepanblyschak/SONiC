# Moving PTF test control flow to pytest proposal

## Motivation

The current way SONiC management infrastructure runs traffic test is:
- sonic-mgmt docker node running ansible/pytest flow:
  - deploy PTF script on PTF host
  - prepare parameters, configuration files for PTF script to run
  - runs PTF script, asserting on return code

The aproach described brings in some difficulties in writing/debugging such tests:

### 1. It is difficult to pass the context to PTF script

Usually this is done via *test prameters* and each PTF script includes a boilerplate code like this:

```python
 self.router_mac = self.test_params['router_mac']
 self.testbed_type = self.test_params['testbed_type']
 self.tor_ports = [int(p) for p in self.test_params['tor_ports'].split(',')]
 # a lot of other test paramters
```

The corresponding command line to run just a PTF script (which is usefull for test developing/debugging purpose) is quite big and inconvinient:

e.g. to run current ACL test

```
ptf --test-dir acstests acltb_test.AclTest   --platform-dir ptftests  --platform remote  -t \"router_mac='98:03:9b:94:d4:80';testbed_type='t1';tor_ports='27,25,26,17,24,31,23,30,19,18,16,22,29,20,28,21';spine_ports='2,0,1,13,14,8,4,9,3,12,11,10,7,15,5,6';dst_ip_tor='172.16.1.0';dst_ip_tor_forwarded='172.16.2.0';dst_ip_tor_blocked='172.16.3.0';dst_ip_spine='192.168.0.0';dst_ip_spine_forwarded='192.168.0.16';dst_ip_spine_blocked='192.168.0.17'\"  --log-file /tmp/acltb_test.AclTest.ingress.2019-07-21-21:37:10.log
```

Moreover, in some cases ansible/pytest has to prepare a whole file to pass to PTF script, so PTF has to parse the file to get neccessary information.
This is also inconvinient as user has to run whole ansible/pytest at least once to generate those files.

In conclusion, the main thought hear is that this approach brings difficulties in writing such tests, as contorlling node has to serialize, pass a test context to PTF scripts, so that PTF scripts can deserialize and run traffic tests accordingly.

### 2. Harder to debug PTF tests

If one wants to debug PTF test cases, one has to:
- stop controlling node before PTF runs
- figure out which parateres are requiered for PTF to run
- run PTF test and set a break point on interested test case code on ptf host

The whole process includes switching between sonic-mgmt node and ptf host.

### 3. Test frameworks inconsistency

PTF is based on UnitTest, while controlling node runs py.test code.
One should be familiar with both to develop traffic tests.

### 4. Inconvinience in running single traffic test case

While reworking tests on py.test we aim to meet two requirements:
 - provide ability to run a single traffic case without running any other test cases.
 - care about test run performance (all current ansible ACL test cases take ~40m)

In current aproach PTF tests are designed such that they have single test class instance which runs all test cases at one run.
In order to meet test cases indenependency we have to rewrite PTF tests and break down a single test class into smaller test classes, each test case run invokes a PTF command line execution on PTF host.
This significantly slows down the test, e.g. for ACL tests:

Currently there is one AclTest object which runs in one command and executes all traffic test cases in ~20 sec.

If we break down AclTest into smaller test cases (SrcIpMatchForward, L4DstPortMatchDropped, etc. ~20 test classes), each test case run from py.test execute PTF command line and the whole test run time increases to ~2 min. (x2 for tor->spine/spine->tor, x4 for different test scenarios, x2 for ingress/egress traffic test). Each new rule and corresponding test case adds ~1.5 min and for ACL rule testing there can be much more than 20 test cases.

## Overview of PTF/py.test integration

The idea is to move all test control flow that currently is written in PTF scripts to py.test. **NOTE** It does not exclude legacy approach, so FDB, BGP speacker tests etc. can still use external PTF scripts.

In order to implement this we would need to run **ptf_nn_agent.py** on PTF host. The sonic-mgmt host will connect to **ptf_nn_agent.py** and control PTF over TCP.

This means that py.test code will prepare packets to send and match and also send and receive using PTF framework functionallity, however actual traffic is sent and received on PTF host.

1. As the traffic test cases require ptf_nn_agent.py to be running on PTF host, this can easily be implementated as py.test fixture.

2. Since lots of testutils functionality depends on PTF test class object which has to be inherited from BaseTest (which inherits from unittest.TestCase) there should be an adapter object which will be used by py.test code.

```python
# 'test' is an instance of ptf.basetest.BaseTest class
def verify_packet_any_port(test, pkt, ports=[], device_number=0):
    # ...
```

In order to provide same interface as BaseTest we will have to implement PtfTestAdapter object:

Example minimal implementation:


```python
# tests/ptf_test_adapter.py
class PtfTestAdapter(ptf.basetest.BaseTest):
    def __init__(self, ptf_ip, port):
        '''
        intiialize PTF parameters, instantiate nn dataplane instance
        '''
        ptf.ptfutils.default_timeout = DEF_TIMEOUT
        ptf.ptfutils.default_negative_timeout = DEF_NEG_TIMEOUT
        ptf.config = {
            'platform': 'nn',
            'device_sockets': [
                (DEVICE_NUM, list(range(PTF_PORTS_NUM)), 'tcp://{}:{}'.format(ptf_ip, port))
            ],
            'relax': True
        }
        ptf.platforms.nn.platform_config_update(ptf.config)
        ptf.dataplane_instance = DataPlane(config=ptf.config)
```

Putting it all togather in one fixture:

```python
@pytest.fixture
def ptfadapter(ansbile_adhoc, testbed):
    # *setup*
    # make sure PTF nn agent is running
    # create PtfTestAdapter
    yield ptf_test_adapter
    # *teardown*
    # deinit ptf data plane instance
    # kill nn agent
```

Usage example:

```python
def test_some_traffic(ptfadapter):
    pkt = testutils.simple_tcp_packet(eth_dst=host_facts['ansible_Ethernet0']['macaddress'],
        eth_src=ptfadapter.dataplane.get_mac(0, 0),
        ip_src='1.1.1.1',
        ip_dst='192.168.0.1',
        ip_ttl=64,
        tcp_sport=1234,
        tcp_dport4321)
    exp_pkt = pkt.copy()
    exp_pkt = mask.Mask(exp_pkt)
    exp_pkt.set_do_not_care_scapy(packet.Ether, 'dst')
    exp_pkt.set_do_not_care_scapy(packet.Ether, 'src')
    exp_pkt.set_do_not_care_scapy(packet.IP, 'ttl')
    exp_pkt.set_do_not_care_scapy(packet.IP, 'chksum')

    testutils.send(ptfadapter, 5, pkt)
    testutils.verify_packet_any_port(ptfadapter, exp_pkt, ports=[28, 29, 30, 31])
```

**NOTE** : despite the fact that PTF testutils uses unittest's methods like *assertTrue*, *fail* etc., py.test has some sort of support for that, so in case when *testutils.verify_packet_any_port* fails by calling UnitTest *fail* it is still correctly handled by py.test:

```
test_l2.py::test_broken FAILED                                                                                                                                                                                                      [100%]

================================================================================================================ FAILURES =================================================================================================================
_______________________________________________________________________________________________________________ test_broken _______________________________________________________________________________________________________________

ptf_adapter = <[AttributeError("'PtfTestAdapter' object has no attribute '_testMethodName'") raised in repr()] SafeRepr object at 0x7fde86de8ef0>

    def test_broken(ptf_adapter):
        pkt = testutils.simple_tcp_packet()
        exp_pkt = pkt.copy()
        exp_pkt.sport = 2355
        testutils.send(ptf_adapter, 1, pkt)
>       testutils.verify_packet(ptf_adapter, exp_pkt, 2)

test_l2.py:21:
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
/usr/local/lib/python2.7/dist-packages/ptf-0.9.1-py2.7.egg/ptf/testutils.py:2405: in verify_packet
    % (device, port, result.format()))
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

msg = 'Expected packet was not received on device 0, port 2.\n========== EXPECTED ==========\ndst        : DestMACField     ...8 29  ...... !"#$%&\'()\n0060  2A 2B 2C 2D                                      *+,-\n==============================\n'

    def fail(self, msg=None):
        """Fail immediately, with the given message."""
>       raise self.failureException(msg)
E       AssertionError: Expected packet was not received on device 0, port 2.
E       ========== EXPECTED ==========
E       dst        : DestMACField                        = '00:01:02:03:04:05' (None)
E       src        : SourceMACField                      = '00:06:07:08:09:0a' (None)
E       type       : XShortEnumField                     = 2048            (36864)
E       --
E       version    : BitField (4 bits)                   = 4               (4)
E       ihl        : BitField (4 bits)                   = None            (None)
E       tos        : XByteField                          = 0               (0)
E       len        : ShortField                          = None            (None)
E       id         : ShortField                          = 1               (1)
E       flags      : FlagsField (3 bits)                 = <Flag 0 ()>     (<Flag 0 ()>)
E       frag       : BitField (13 bits)                  = 0               (0)
E       ttl        : ByteField                           = 64              (64)
E       proto      : ByteEnumField                       = 6               (0)
E       chksum     : XShortField                         = None            (None)
E       src        : SourceIPField                       = '192.168.0.1'   (None)
E       dst        : DestIPField                         = '192.168.0.2'   (None)
E       options    : PacketListField                     = []              ([])
E       --
E       sport      : ShortEnumField                      = 2355            (20)
E       dport      : ShortEnumField                      = 80              (80)
E       seq        : IntField                            = 0               (0)
E       ack        : IntField                            = 0               (0)
E       dataofs    : BitField (4 bits)                   = None            (None)
E       reserved   : BitField (3 bits)                   = 0               (0)
E       flags      : FlagsField (9 bits)                 = <Flag 2 (S)>    (<Flag 2 (S)>)
E       window     : ShortField                          = 8192            (8192)
E       chksum     : XShortField                         = None            (None)
E       urgptr     : ShortField                          = 0               (0)
E       options    : TCPOptionsField                     = []              ('')
E       --
E       load       : StrField                            = '\x00\x01\x02\x03\x04\x05\x06\x07\x08\t\n\x0b\x0c\r\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f !"#$%&\'()*+,-' ('')
E       --
E       0000  00 01 02 03 04 05 00 06 07 08 09 0A 08 00 45 00  ..............E.
E       0010  00 56 00 01 00 00 40 06 F9 4D C0 A8 00 01 C0 A8  .V....@..M......
E       0020  00 02 09 33 00 50 00 00 00 00 00 00 00 00 50 02  ...3.P........P.
E       0030  20 00 08 CB 00 00 00 01 02 03 04 05 06 07 08 09   ...............
E       0040  0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19  ................
E       0050  1A 1B 1C 1D 1E 1F 20 21 22 23 24 25 26 27 28 29  ...... !"#$%&'()
E       0060  2A 2B 2C 2D                                      *+,-
E       ========== RECEIVED ==========
E       1 total packets. Displaying most recent 1 packets:
E       ------------------------------
E       dst        : DestMACField                        = '00:01:02:03:04:05' (None)
E       src        : SourceMACField                      = '00:06:07:08:09:0a' (None)
E       type       : XShortEnumField                     = 2048            (36864)
E       --
E       version    : BitField (4 bits)                   = 4               (4)
E       ihl        : BitField (4 bits)                   = 5               (None)
E       tos        : XByteField                          = 0               (0)
E       len        : ShortField                          = 86              (None)
E       id         : ShortField                          = 1               (1)
E       flags      : FlagsField (3 bits)                 = <Flag 0 ()>     (<Flag 0 ()>)
E       frag       : BitField (13 bits)                  = 0               (0)
E       ttl        : ByteField                           = 64              (64)
E       proto      : ByteEnumField                       = 6               (0)
E       chksum     : XShortField                         = 63821           (None)
E       src        : SourceIPField                       = '192.168.0.1'   (None)
E       dst        : DestIPField                         = '192.168.0.2'   (None)
E       options    : PacketListField                     = []              ([])
E       --
E       sport      : ShortEnumField                      = 1234            (20)
E       dport      : ShortEnumField                      = 80              (80)
E       seq        : IntField                            = 0               (0)
E       ack        : IntField                            = 0               (0)
E       dataofs    : BitField (4 bits)                   = 5               (None)
E       reserved   : BitField (3 bits)                   = 0               (0)
E       flags      : FlagsField (9 bits)                 = <Flag 2 (S)>    (<Flag 2 (S)>)
E       window     : ShortField                          = 8192            (8192)
E       chksum     : XShortField                         = 3372            (None)
E       urgptr     : ShortField                          = 0               (0)
E       options    : TCPOptionsField                     = []              ('')
E       --
E       load       : StrField                            = '\x00\x01\x02\x03\x04\x05\x06\x07\x08\t\n\x0b\x0c\r\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f !"#$%&\'()*+,-' ('')
E       --
E       0000  00 01 02 03 04 05 00 06 07 08 09 0A 08 00 45 00  ..............E.
E       0010  00 56 00 01 00 00 40 06 F9 4D C0 A8 00 01 C0 A8  .V....@..M......
E       0020  00 02 04 D2 00 50 00 00 00 00 00 00 00 00 50 02  .....P........P.
E       0030  20 00 0D 2C 00 00 00 01 02 03 04 05 06 07 08 09   ..,............
E       0040  0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 19  ................
E       0050  1A 1B 1C 1D 1E 1F 20 21 22 23 24 25 26 27 28 29  ...... !"#$%&'()
E       0060  2A 2B 2C 2D                                      *+,-
E       ==============================

/usr/lib/python2.7/unittest/case.py:410: AssertionError

```

Also logging from PTF dataplane is correctly captured by pytest:

```
WARNING  dataplane:dataplane.py:873 Dataplane poll with exp_pkt but no port number
...
INFO     dataplane:dataplane.py:634 Thread exit
```

One more benefit is that PTF logs will be on sonic-mgmt node and not on PTF node which can be destroyed/recreated and logs can be lost.

The above described method of running traffic tests seems to solve all above issues and meet all requirements:

- there is no passing test information/context to PTF via command line or some files
- it is easy to debug by just setting a breakpoint in interested place in pytest
- no boilerplate code in PTF scripts that deserialize command line input
- easy to use *ptfadpater* fixture to pass in to PTF testutils functions
- logs/asserts from PTF framework are captured by py.test framework without additional integration
- no seperate PTF invokations per each test case which improves performance
- easy to seperate test cases directly in py.test code
