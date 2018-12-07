[>>BACK TO MAIN MENU](https://aizuddin85.github.io/)

# __OCP 3.11: RECEIVING ALERT FROM PROMETHEUS (with Slack Integration)__
This wiki consists of two major sections:  
1. Simple Alerting Test using '*DeadMansSwitch*'.
2. Configure configMap to create test trigger.


## __SIMPLE ALERTING TEST USING '*DeadMansSwitch*'__

### __1. INTRODUCTION__

In order to receive alert from openshift-monitoring, one need to configure alertmanager route for alertname that he or she wanted to received for. In this case, we going to send all default alertname = DeadMansSwitch to Slack channel.

DeadMansSwitch is one of the default general.rules that to ensure that the entire Alerting pipeline is working. Hence expect to received this notification from time to time (configurable delay). When it stopped notified, then high probably something is not working when alert are being sent.

#### __1.2 HIGH-LEVEL STEPS__

In bird eye view, the steps are:

1. <a href="https://api.slack.com/incoming-webhooks" target="_blank">Install and Enable Slack incoming WebHook</a>

2. Configure secret for AlertManager & Test trigger


#### __1.3 CONFIGURATION STEPS__

#### __1.3.1 CONFIGURATION STEPS: Install and Enable Slack incoming WebHook__

1. Install and enable Slack incoming webhook plugin.

![alt text](https://aizuddin85.github.io/prometheus_alert/images/enable_incoming_wh.png "enable-incoming-webhook-plugin")

2. Create an Slack incoming webhook.

Login to slack, and create incoming webhook. Click 'Add Configuration' and then 'Add Incoming WebHooks integration'. Select appropriate channel for your webhook.

![alt text](https://aizuddin85.github.io/prometheus_alert/images/create_incoming_wh.png "create_incoming_wh")

Once webhook succesfully created below page will be shown:


![alt text](https://aizuddin85.github.io/prometheus_alert/images/create_wh.png "create_wh")

3. Now we can test our webhook.

There will be an cURL example to test that should return 'ok' and message displayed on the slack channel room.

![alt text](https://aizuddin85.github.io/prometheus_alert/images/example_curl_wh.png "example_curl_wh")

This conclude that the incoming webhook is working as expected.


4. Export and save current secret.
```
[root@bastion 3.11]# oc export secret alertmanager-main -n openshift-monitoring > alertmanager-main.yaml
Command "export" is deprecated, use the oc get --export
```

5. Create the new alertmanager-main.yaml that include slack plugin.
```
[root@bastion 3.11]# cat alertmanager.yaml 
global:
  resolve_timeout: 5m
route:
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: default
  routes:
  - match:
      alertname: DeadMansSwitch
    repeat_interval: 5m
    receiver: slack_general
receivers:
- name: default
- name: deadmansswitch
- name: slack_general
  slack_configs:
  - api_url: https://hooks.slack.com/services/TEKMSBUKD/BEM7BV3RC/XXXXXXXXXX
    channel: '#general'
```

6. Delete and recreate the secret.
```
[root@bastion 3.11]# oc delete secret alertmanager-main -n openshift-monitoring
secret "alertmanager-main" deleted
[root@bastion 3.11]# oc create secret generic alertmanager-main --from-file=alertmanager.yaml -n openshift-monitoring
secret/alertmanager-main created
```

7. The alertmanager now will reload this new secret.

![alt text](https://aizuddin85.github.io/prometheus_alert/images/alertmanager_logs_reload.png "alertmanager_logs_reload")

8. The configuration from alertmanager look like this now:  
https://alertmanager-main-openshift-monitoring.cloudapps.example.com/#/status 

![alt text](https://aizuddin85.github.io/prometheus_alert/images/alertmanager_config.png	"alertmanager_config")

9. From the Slack channel, there will be message displayed:

![alt text](https://aizuddin85.github.io/prometheus_alert/images/dms_verify.png	"dms_verify")


#### __1.3.2 CONFIGURATION STEPS:  Configure secret for AlertManager & Test trigger__

At this stage, we have test alerting pipeline from Prometheus to Slack using incoming webhook. Now we going to test actual operation of the Prometheus monitoring and alerting.

As a test case, we are going:

1. Edit custom resource definition(CRD) of the Prometheus rule so that:

   - any node that has pod running more than 5 pods are in the critical state

2. Fire alert if condition above stay more than 2 minutes.

__*PRE-REQ*__

Reconfigure alertmanager.yaml as below using above section guide:
```
global:
  resolve_timeout: 5m
route:
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: default
  routes:
  - match:
      alertname: DeadMansSwitch
    repeat_interval: 5m
    receiver: deadmansswitch
receivers:
- name: default
  slack_configs:
  - api_url: https://hooks.slack.com/services/TEKMSBUKD/BEM7BV3RC/XXXXXXX
    channel: "#general"
- name: deadmansswitch
```
__STEPS__:
1. Create a new CRD for prometheus-k8s-rules.
```
[root@master01 ~]# oc get prometheusrule prometheus-k8s-rules -o yaml > prometheus-k8s-rules.yaml
```
2. Change the definition of the alert below

Before:
```
---- TRUNCATED ----
  
  - alert: KubeletTooManyPods
      annotations:
        message: Kubelet {{$labels.instance}} is running {{$value}} pods, close to
          the limit of 110.
      expr: |
        kubelet_running_pod_count{job="kubelet"} > 100
      for: 15m

---- TRUNCATED ----
```

After:
```
---- TRUNCATED ----
  
  - alert: KubeletTooManyPods
      annotations:
        message: Kubelet {{$labels.instance}} is running {{$value}} pods, close to
          the limit of 110.
      expr: |
        kubelet_running_pod_count{job="kubelet"} > 10
      for: 2m

---- TRUNCATED ----
```

3. Delete current prometheus-k8s-rules.

```
[root@master01 ~]# oc delete prometheusrule prometheus-k8s-rules
prometheusrule.monitoring.coreos.com "prometheus-k8s-rules" deleted
```

4. Create new edited rules.
```
[root@master01 ~]# oc create -f prometheus-k8s-rules.yaml
prometheusrule.monitoring.coreos.com/prometheus-k8s-rules created
[root@master01 ~]# 
```


 5. Edit the prometheus-k8s-rulefiles-0 configMap and **removed** below labels.

**NOTE**: This is necessary due to whatever changes to the prometheusrule and configMap will be reconciled by prometheus config/configmap reloader to default value if these labels exists. Current 3.11.43-1 version does not support custom rules. <a href="https://access.redhat.com/documentation/en-us/openshift_container_platform/3.11/html/configuring_clusters/prometheus-cluster-monitoring#alerting-rules" target="_blank">[1]</a>

Doing this will break Prometheus reloader. Do this ONLY on testing system.
```
  labels:
    managed-by: prometheus-operator
    prometheus-name: k8s
```

6. Observe https://prometheus-k8s-openshift-monitoring.cloudapps.example.com.my/alerts. The detection is now in state pending, counter are ticking for 2m.

![alt text](https://aizuddin85.github.io/prometheus_alert/images/alert_prometheus.png	"alert_prometheus")


7. Once the timer timed out, we can see the alert being sent to Slack channel room.

![alt text](https://aizuddin85.github.io/prometheus_alert/images/alert_received.png	"alert_received")

And also from the alert status page:

![alt text](https://aizuddin85.github.io/prometheus_alert/images/alert_status.png	"alert_status")

## REFERENCES

[1.alertmanager/simple.yml at master · prometheus/alertmanager · GitHub ](https://github.com/prometheus/alertmanager/blob/master/doc/examples/simple.yml)

[2. Configuring Prometheus Alertmanager ](https://coreos.com/tectonic/docs/latest/tectonic-prometheus-operator/user-guides/configuring-prometheus-alertmanager.html)

[3. https://prometheus.io/docs/alerting/configuration/ ](https://prometheus.io/docs/alerting/configuration/)

[>>BACK TO MAIN MENU](https://aizuddin85.github.io/)