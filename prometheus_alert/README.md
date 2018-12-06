# __OCP 3.11: RECEIVING ALERT FROM PROMETHEUS (with Slack Integration)__
This wiki consists of two major sections:  
1. Simple Alerting Test using '*DeadMansSwitch*'.
2. Configure configMap to create test trigger.


## __SIMPLE ALERTING TEST USING '*DeadMansSwitch*'__

### 1. INTRODUCTION

In order to receive alert from openshift-monitoring, one need to configure alertmanager route for alertname that he or she wanted to received for. In this case, we going to send all default alertname = DeadMansSwitch to Slack channel.

DeadMansSwitch is one of the default general.rules that to ensure that the entire Alerting pipeline is working. Hence expect to received this notification from time to time (configurable delay). When it stopped notified, then high probably something is not working when alert are being sent.

#### 1.2 HIGH-LEVEL STEPS

In bird eye view, the steps are:

1. [Install and Enable Slack incoming WebHook](https://api.slack.com/incoming-webhooks)

2. Configure secret for AlertManager


#### 1.3 CONFIGURATION STEPS

1. Install and enable incoming webhook.

![alt text](https://aizuddin85.github.io/prometheus_alert/images/enable_incoming_wh.png "enable-incoming-webhook-plugin")