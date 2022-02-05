# Noti Event Types

Collection of ready-to-use event types for the "Noti - Activity Notifications WordPress" plugin

## Prerequisites

The easiest way to start using these event types is to install WordPress plugin [Noti - Activity Notifications](https://wordpress.org/plugins/noti-activity-notification).

## Configurations Overview

Event type configuration is based on JSON format and contains only few key properties. Under the hood it is a fork of [JSON Policy](https://jsonpolicy.github.io/) open-source project that I maintain. So let's take a simple example and digest it in greater detail.

```json
{
    "Event": "filter:pre_update_site_option_auto_update_themes",
    "Level": "notice",
    "RequiredVersion": "WordPress 3.7.0+",
    "Metadata": {
        "user_id": "${USER.ID}",
        "user_ip": "${USER.ip}",
        "state": "${HTTP_POST.state}",
        "theme": "${FUNC.wp_get_theme(HTTP_POST.asset).name}"
    },
    "Condition": {
        "Equals": {
            "${HTTP_POST.action}": "toggle-auto-updates",
            "${HTTP_POST.type}": "theme"
        }
    },
    "MessageMarkdown": "**${FUNC.get_userdata(EVENT_META.user_id).display_name}** toggled ${EVENT_META.state} auto-update for the **${EVENT_META.theme}** theme",
    "Notifications": [
        {
            "Type": "email"
        }
    ]
}
```

The `Event` property declares what type of hook we are going to be listening. It starts with either `filter:` or `action:` following by the name of the hook (in our example it is `pre_update_site_option_auto_update_themes`). WordPress core has hundreds of hooks you can "subscribe" to and most of mature plugins or themes also introduce their own hooks.

> FYI! It is important to pay attention to type of hook and correctly state it for the `Event` property. WordPress filters always return a value, while actions do not. That is why if you made a mistake instead of `filter:` used `action:`, you most likely will break some functionality.

The `Level` property is optional and can be used to declare the "level of event importance". For example, if user modified some plugin's file, you probably want to consider this type of event as "Critical". However, if draft post is modified, this may not be as important and can be labeled as "Notice". It is up to you and your business nature to define what is more or less important. This information is used only for your convenience to better work with large list of events in the log.

The `RequiredVersion` property is also optional, however, can be a critical piece of information to let you know what is the minimum required version of software you need so you event type can function properly. Some hooks may not be available or have different set of parameter in earlier version.

The `Metadata` is a map of key/value pairs that declares what needs to be captured and stored in the database as metadata when event is triggered. This information is stored in the `noti_eventmeta` database table. You have full freedom to capture as much information as needed and below I'll explain in more detail about the concept of markers (e.g. `${USER.ID}`, `${FUNC.wp_get_theme(HTTP_POST.asset).name}`).

The `Condition` is where the great power of these configurations come from. Basically here you can defined set of conditions under which event should be captured. For example, you can define a custom event type that captures when ONLY users with subscriber role posted a comment, or when any post in "Science" category was modified. There is no limit what you can do here. For more information about conditions, you can refer to my documentation for the [JSON Policy Conditions](https://jsonpolicy.github.io/overview/policy-condition.html).

The `MessageMarkdown` is optional, however, useful way to construct your own custom message that you'll see on the "Activity Log" page inside your WordPress dashboard. This message property is also used to compile the message that is send as notification to your email box. It heavily relies on the information that is captured when event occurred, so if you need to be more verbose with your message, you probably would want to declare more key/value pairs in the `Metadata` property.

Last, but not least, the `Notifications` contains the array of notification types that will be triggered to route captured events to  different destination(s). In the initial Noti version, to avoid performance issues, events are captured in the database first, and then routed to elsewhere with WordPress cron job that occurs every minute. This behavior may be changed in later Noti versions if more near-to-realtime notifications will be needed. A bit more information about currently supported notification types you can find below.

## Configuration Markers

Marker (or some may think about it as token) is a piece of configuration that is replaced with some value at runtime (at the time the event is triggered or notification is prepared to be sent). This functionality is based on the [JSON Policy Marker](https://jsonpolicy.github.io/overview/policy-marker.html) concept and you can learn a bit more about it by following the mentioned link.

Below is the list of currently supported markers and how they can be used:

* `${ARRAY_MAP.functionName(arrayArg)}` takes that `arrayArg` value (which is expected to be and array) and iterate over each element of it by applying function `functionName`. For example `${ARRAY_MAP.get_plugin_data(HTTP_POST.checked).Name}` expects that there is an array of string values in the incoming `$_POST[checked]`, and we prepare the list of plugins names based on these values.
* `${FUNC.functionName(...args)}` executes callable `functionName` by passing one or more arguments in it.
* `${CONST.constantName}` if constant is defined, get its value.
* `${USER.propertyName}`, get current user's property. Basically anything that is available in the `WP_User` object. Additionally, you can also fetch `ip`, `ipAddress`, `authenticated`, `isAuthenticated`, `capabilities`, `caps`.
* `${USER_OPTION.optionName}` get current user's option. This invokes WordPress core [get_user_option](https://developer.wordpress.org/reference/functions/get_user_option/) function.
* `${USER_META.metaName}` get current user's meta data. This invokes WordPress core [get_user_meta](https://developer.wordpress.org/reference/functions/get_user_meta/) function.
* `${WP_OPTION.optionName}` get current site option. This invokes WordPress core [get_option](https://developer.wordpress.org/reference/functions/get_option/) function or [get_site_option](https://developer.wordpress.org/reference/functions/get_site_option/) if multisite.
* `${WP_SITE.siteDetail}` get current site detail. This invokes WordPress core [get_blog_details](https://developer.wordpress.org/reference/functions/get_blog_details/) function.
* `${PHP_GLOBAL.variableName}` get globally available in the `$_GLOBALS` variable's value.
* `${LISTENER.metaName}`
* `${EVENT_META.metaName}`
* `${EVENT.propertyName}`
* `${EVENT_TYPE.propertyName}`
* `${EVENT_TYPE_META.metaName}`

## Notification Types

TODO

## Contributing To The Project
To contribute to the project, please follow these steps:

1. Fork this repository.
2. Create a branch: `git checkout -b <branch_name>`.
3. Make your changes and commit them: `git commit -m '<commit_message>'`
4. Push to the original branch: `git push origin <project_name>/<location>`
5. Create the pull request.

Alternatively see the GitHub documentation on [creating a pull request](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/creating-a-pull-request).


## Contact

If you want to contact me you can reach me at <vasyl@vasyltech.com>.


## License

This project uses the following license: [<GNU General Public License>](https://www.gnu.org/licenses/#GPL).