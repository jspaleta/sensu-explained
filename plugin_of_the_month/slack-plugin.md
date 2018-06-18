## Sensu-Plugins-Slack

![Alt text](images/slack_handler.png)

Easily send formatted event notifications to one or more more slack channels using Slack's [webhook](https://api.slack.com/incoming-webhooks) framework with a simple handler configuration. Take more control by crafting interactive slack messages using custom ERB templates.

References:

* [Plugin Repository](https://github.com/sensu-plugins/sensu-plugins-slack/)
* [Gem Homepage](https://rubygems.org/gems/sensu-plugins-slack)

### Prerequisites

#### Plugin Installation
You'll want to make sure your sensu server has the sensu-plugins-slack ruby gem installed. If you are using the embedded ruby environment provided by the official Sensu 1.x packages, all you can use `sensu-install -p slack` to install the plugin into the embedded ruby  environment.  If you are using a different ruby environment, you can use `gem install sensu-plugins-slack` to install the gem.

#### Create Slack App with Incoming Webhook
The examples below all make use of Slack's incoming webhook url feature. You can follow Slack's [tutorial](https://api.slack.com/incoming-webhooks) for creating your own Slack App with incoming webhook url enabled. Once you have a Slack App with an incoming webhook url enabled you'll be able to use that webhook url to write messsages into the channels associated with the webhook definition. Please note, you'll be required to register a different webhook url for each channel you want to send messages to.

You'll need to take the webhook url and place it into a json configuration located into `/etc/sensu/conf.d/`.

Here is an example of a basic test configuration that we'll be building on for the rest of the walk through examples.
```
{
  "slack_test": {
    "webhook_url": "https://hooks.slack.com/services/something/something/something",
  }
}

```
You'll need to replace the webhook_url value with the correct webhook url that you created for your slack channel.


### Plugin Executables

#### handler-slack.rb
This ruby executable is usable as a Sensu 1.x pipe handler and allows you to send an alert to a single slack channel. By default this command will look for configuration under the `slack` settings scope. This can be changed using the `-j` argument. 


Here is a handler definition that makes use of the minimal slack_test configuration defined above.
```
{
  "handlers": {
    "slack_test_handler": {
      "type": "pipe",
      "command": "handler-slack.rb -j slack_test"
    }
  }
}

```

If you have your executable environment configured to include the path in which the plugin executables were installed, you can test this minimal configuration by executing the handler on the commandline by injecting the json representation of a sensu event via a pipe.

```
echo '{ "check" : { "output" : "test warning output", "name" : "slack_test_check", "status" : 1} , "client" : {"address" : "unknown","subscriptions" : ["slack_test"], "name" : "slack_test_client" } }' | handler-slack.rb -j slack_test
```

Running this echo pipe should result in a warning alert message being sent into the channel registered with the incoming webhook url defined in the `slack_test` configuration file. The warning message uses default formatting, most of which can be redefined using optional configuration parameters or a template file.


#### handler-slack-multichannel.rb
This ruby executable works similarly to the single channel variant but adds the ability to define multiple channels to send alerts to, each using a different webhook url. The multichannel handler requires a little more configuration to be usable.  Here is a simple example of the multichannel configuration:

```
{
  "slack_multi_test": {
    "webhook_urls": {
      "test-one":   "https://hooks.slack.com/services/AAAAAA",
      "test-two":   "https://hooks.slack.com/services/BBBBBB",
      "test-three": "https://hooks.slack.com/services/CCCCCC",
      "test-four":  "https://hooks.slack.com/services/DDDDDD"
    },
    "channels": {
      "default":    [ "test-one" ],
      "compulsory": [ "test-two" ]
    }
  }
}

```
The `webhook_urls` hash defines key/value pairs, where each key is a a slack channel name and the value is the corresponding webhook url. The `channels` attribute defines an array of default channels to use as a fallback, as well as an array of channels to send to all alerts to. You'll want to define a default array of channels to use as a fallback if no channels are defined as part of the client or check configurations.

The multichannel handler will prefer channels defined at the check scope over the client scope over the default channels. Here try these cmdline tests with the test configuration above.

```
echo '{ "check" : { "output" : "test warning output", "name" : "slack_test_check", "status" : 1, "slack" : { "channels" : ["test-four"] }} , "client" : {"address" : "unknown","subscriptions" : ["slack_test"], "name" : "slack_test" ,"slack" : { "channels" : ["test-three"] }} }' | handler-slack-multichannel.rb -j slack_multi_test
```

This command will result in messages being sent to channels:

* `test-four`: defined in the `check` scope
* `test-two`: defined as `compulsory` in the `slack_multi_test` configuration



### Advanced Configuration Options
* **dashboard**: publicly accessible dashboard url stub for event alert. If defined, the pre-formatted Slack message will include a link to the dashboard. Uchiwa compatible values take the form: `http(s)://<host>:<port>/#/client/<DataCenter>/`
* **message_prefix**: static text to prepend to pre-formatted event notices sent by the slack handler. 
* **fields**: Array of client attributes to include in the pre-formatted message
* **message_template**: ERB template file for message content. This template will have access to the `@event` variable, a ruby hash object representation of the Sensu event data passed to the handler executable via stdlin. `@event['client']` and `@event['check']` hold the corresponding client and check information.
* * **payload_template**: ERB template file for the Slack payload. If this template option listed above will be overridden.

* _HTTP Proxy_:  post message to a http proxy via `Net::HTTP::Proxy`
    * **proxy_address**: proxy address 
    * **proxy_port**: proxy port 
    * **proxy_username**: proxy username 
    * **proxy_passord**: proxy password 





