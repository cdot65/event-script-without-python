# Python-less ping test failed event script

[![N|Solid](https://upload.wikimedia.org/wikipedia/commons/3/31/Juniper_Networks_logo.svg)](https://www.juniper.net/us/en.html)

## Overview

This repository will provide a Python-less Event Script for Juniper Networks devices running JunOS.

This example creates an event policy named `disable-interface-on-ping-failure`. The event policy is configured so that the *eventd* process listens for *PING_TEST_FAILED* events generated by a specific RPM probe and associated with the *ge-0/0/4* interface.

If three *PING_TEST_FAILED* events occur for the given interface within a 60-second interval, the event policy executes a change configuration action. The event policy configuration commands administratively disable the interface.

To test the event policy, the example configures an RPM probe that pings the IP address associated with the *ge-0/0/4* interface. In this example, the *ge-0/0/4.0* interface is configured with the IPv4 address `172.20.12.0/26`.

By default, one probe is sent per test, and the example uses a 5-second pause between tests. After three successive probe losses, the RPM probe generates a *PING_TEST_FAILED* event.

Because multiple RPM tests could be running simultaneously, the event policy matches the owner-name and test-name attributes of the received *PING_TEST_FAILED* events to the RPM probe owner name and test name.

When the RPM probe generates three *PING_TEST_FAILED* events in a 60-second interval, it triggers the event policy, which disables the interface.

This event policy also demonstrates how to restrict the execution of the same configuration change multiple times because of occurrences of the same event or correlated events. In this example, the within 60 trigger on 3 statement specifies that the configuration change is only triggered on the third occurrence of a *PING_TEST_FAILED* event within a 60-second interval. The trigger until 4 statement specifies that subsequent occurrences of the *PING_TEST_FAILED* event should not cause the event policy to re-trigger.

NOTE: If you only configure the trigger on 3 condition, the commit operation might go into a loop. The combination of trigger on 3 and trigger until 4 prevents the event policy from repeatedly making the same configuration change.

## ⚙️ Configuration

### Configuring the RPM probe

1. Create an RPM probe named PING_PE1 with owner PE_HEALTH
2. Configure the RPM probe to send the ICMP echo requests to the *ge-0/0/4* interface at IP address `172.20.12.0`.
3. Configure a 5-second pause between test windows.
4. Configure the RPM probe threshold so that the *PING_TEST_FAILED* event is triggered after three successive probe losses.
5. Configure a new log file at the `[edit system syslog]` hierarchy level to record syslog events of facility daemon and severity info. This captures the events sent during the probe tests.

#### Set RPM configuration commands

``` yaml
set services rpm probe PE_HEALTH test PING_PE1 probe-type icmp-ping
set services rpm probe PE_HEALTH test PING_PE1 target address 172.20.12.0
set services rpm probe PE_HEALTH test PING_PE1 test-interval 5
set services rpm probe PE_HEALTH test PING_PE1 thresholds successive-loss 3
set system syslog file syslog-event-daemon-info daemon info
```

#### Resulting RPM configuration

``` yaml
root@ce1# show | compare
[edit system syslog]
+    file syslog-event-daemon-info {
+        daemon info;
+    }
[edit services]
+   rpm {
+       probe PE_HEALTH {
+           test PING_PE1 {
+               probe-type icmp-ping;
+               target address 172.20.12.0;
+               test-interval 5;
+               thresholds {
+                   successive-loss 3;
+               }
+           }
+       }
+   }
```

### Configuring the Event Policy

1. Create and name the event-policy.
2. Configure the policy to trigger when three *PING_TEST_FAILED* events occur within 60 seconds.
3. Configure the `within 65 trigger until 4` statement to prevent the policy from re-triggering if more than three *PING_TEST_FAILED* events occur.
4. Configure the attributes-match statement so that the event policy is only triggered by the *PING_TEST_FAILED* events generated by the associated RPM probe.
5. Specify the configuration mode commands that are executed if the event policy is triggered.
6. Configure each command on a single line, enclose the command string in quotes, and specify the complete statement path.
7. Configure the log option with a comment describing the configuration changes.
8. The comment is added to the commit logs after a successful commit operation is made through the associated event policy.
9. *(Optional)* If you have dual Routing Engines, configure the synchronize option to commit the configuration on both Routing Engines.
Include the `force` option to force the commit on the other Routing Engine, ignoring any warnings. This example does not configure the synchronize and force options.
10. *(Optional)* Configure the username under whose privileges the configuration changes and commit are made.
*If you do not specify a username, the action is executed as user `root`.*

#### Set Event Options configuration commands

``` yaml
set event-options policy DISABLE_ON_PING_FAILURE events PING_TEST_FAILED
set event-options policy DISABLE_ON_PING_FAILURE within 60 trigger on
set event-options policy DISABLE_ON_PING_FAILURE within 60 trigger 3
set event-options policy DISABLE_ON_PING_FAILURE within 65 trigger until
set event-options policy DISABLE_ON_PING_FAILURE within 65 trigger 4
set event-options policy DISABLE_ON_PING_FAILURE attributes-match PING_TEST_FAILED.test-owner matches PE_HEALTH
set event-options policy DISABLE_ON_PING_FAILURE attributes-match PING_TEST_FAILED.test-name matches PING_PE1
set event-options policy DISABLE_ON_PING_FAILURE then change-configuration commands "set interfaces ge-0/0/4 disable"
set event-options policy DISABLE_ON_PING_FAILURE then change-configuration user-name awx
set event-options policy DISABLE_ON_PING_FAILURE then change-configuration commit-options log "updating configuration from event policy DISABLE_ON_PING_FAILURE"
```

#### Resulting Event Options configuration

``` yaml
root@ce1# show | compare
[edit]
+  event-options {
+      policy DISABLE_ON_PING_FAILURE {
+          events PING_TEST_FAILED;
+          within 60 {
+              trigger on 3;
+          }
+          within 65 {
+              trigger until 4;
+          }
+          attributes-match {
+              PING_TEST_FAILED.test-owner matches PE_HEALTH;
+              PING_TEST_FAILED.test-name matches PING_PE1;
+          }
+          then {
+              change-configuration {
+                  commands {
+                      "set interfaces ge-0/0/4 disable";
+                  }
+                  user-name awx;
+                  commit-options {
+                      log "updating configuration from event policy DISABLE_ON_PING_FAILURE";
+                  }
+              }
+          }
+      }
+  }
```

## 🚀 Execution Verification

### Verifying the Events

To manually test the event policy, take the *ge-0/0/4* interface offline until three *PING_TEST_FAILED* events are generated.

Review the configured syslog file. Verify that the RPM probe generates a *PING_TEST_FAILED* event after successive lost probes.

``` yaml
root@ce1> show log syslog-event-daemon-info
Dec 20 10:30:01  ce1 rmopd[11546]: PING_TEST_COMPLETED: pingCtlOwnerIndex = PE_HEALTH, pingCtlTestName = PING_PE1
Dec 20 10:30:32  ce1 last message repeated 6 times
Dec 20 10:32:09  ce1 last message repeated 19 times
Dec 20 10:32:17  ce1 rmopd[11546]: PING_TEST_FAILED: pingCtlOwnerIndex = PE_HEALTH, pingCtlTestName = PING_PE1
Dec 20 10:32:33  ce1 last message repeated 2 times
Dec 20 10:32:36  ce1 file[54204]: UI_COMMIT_NO_MASTER_PASSWORD:  : No 'system master-password' set
...
Dec 20 10:32:40  ce1 chassisd[9579]: CHASSISD_PARSE_COMPLETE: Using new configuration
Dec 20 10:32:40  ce1 rpd[9722]: RPD_IFL_NOTIFICATION: EVENT [UpDown] ge-0/0/4.0 index 339 [Broadcast Multicast] address #0 50.0.0.10.0.6
Dec 20 10:32:40  ce1 mib2d[9585]: SNMP_TRAP_LINK_DOWN: ifIndex 531, ifAdminStatus down(2), ifOperStatus down(2), ifName ge-0/0/4
Dec 20 10:32:40  ce1 rpd[9722]: RPD_IFA_NOTIFICATION: EVENT <UpDown> ge-0/0/4.0 index 339 172.20.12.3/31 -> zero-len <Broadcast Multicast Localup>
Dec 20 10:32:40  ce1 rpd[9722]: RPD_IFD_NOTIFICATION: EVENT <UpDown> ge-0/0/4 index 153 <Broadcast Multicast> address #0 50.0.0.10.0.6
Dec 20 10:32:40  ce1 rpd[9722]: RPD_RT_HWM_NOTICE: New FIB highwatermark for routes: 4 [2021-12-20 10:32:40]
Dec 20 10:32:40  ce1 l2cpd[9741]: JTASK_TASK_REINIT: Reinitializing
Dec 20 10:32:40  ce1 l2cpd[9741]: L2CPD: SNMP Filter Interface configuration success
Dec 20 10:32:40  ce1 l2cpd[9741]: LLDP_NEIGHBOR_DOWN: A neighbor of interface ge-0/0/4 has gone down. Now, this interface has 0 neighbor/s.
```

### Verifying the Commit

Verify that the event policy commit operation was successful by reviewing the commit log and the messages log file.

Issue the show system commit operational mode command to view the commit log.

``` yaml
root@ce1> show system commit
root@ce1# run show system commit
0   2021-12-20 10:32:40 UTC by awx via junoscript
    updating configuration from event policy DISABLE_ON_PING_FAILURE
1   2021-12-20 10:28:35 UTC by root via cli
2   2021-12-14 16:29:11 UTC by root via cli
```

Review the messages log file.

``` yaml
root@ce1> show log messages | last 20
Dec 20 10:32:36  ce1 file[54204]: UI_COMMIT: User 'awx' requested 'commit' operation (comment: updating configuration from event policy DISABLE_ON_PING_FAILURE)
Dec 20 10:32:40  ce1 ffp: "dynamic-profiles": No change to profiles
Dec 20 10:32:40  ce1 mib2d[9585]: SNMP_TRAP_LINK_DOWN: ifIndex 531, ifAdminStatus down(2), ifOperStatus down(2), ifName ge-0/0/4
Dec 20 10:32:40  ce1 rpd[9722]: RPD_RT_HWM_NOTICE: New FIB highwatermark for routes: 4 [2021-12-20 10:32:40]
Dec 20 10:32:40  ce1 kernel: kresource_ifd_get_ifdslot: phy_ifd is null for kresource key:0
Dec 20 10:32:40  ce1 file[54204]: UI_COMMIT_COMPLETED:  : commit complete
Dec 20 10:32:40  ce1 eventd: EVENTD_CONFIG_CHANGE_SUCCESS: Configuration change successful: while executing policy DISABLE_ON_PING_FAILURE with user awx privileges
```

The output from the show system commit operational mode command and the messages log file verify that Junos OS executed the configured event policy action to modify and commit the configuration. The commit operation, which was made through the event policy under the privileges of the user awx at the given date and time, was successful.

The `show system commit output` and messages log file reference the commit comment specified in the log statement at the `[edit event-options policy disable-interface-on-ping-failure then change-configuration commit-options]` hierarchy level.

### Verifying the Configuration Changes

Verify the configuration changes by reviewing the `[edit interfaces ge-0/0/4]` hierarchy level of the configuration.

``` yaml
root@ce1> show configuration interfaces ge-0/0/4
disable;
unit 0 {
    family inet {
        address 172.20.12.0/26;
    }
}
```

The *ge-0/0/4* configuration hierarchy was modified through the event policy to add the disable statement.

### Verifying the Status of the Interface

Review the output of the show interfaces *ge-0/0/4* operational mode command after the configuration change takes place.

Issue the show interfaces *ge-0/0/4* operational mode command. After the event policy configuration change action disables the interface, the status changes from "Enabled" to "Administratively down".

``` yaml
root@ce1> show interfaces ge-0/0/4
Physical interface: ge-0/0/4, Administratively down, Physical link is Down
  Interface index: 142, SNMP ifIndex: 531
```
