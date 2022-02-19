# Noti Event Types

Collection of ready-to-use event types for the "Noti - Activity Notifications WordPress" plugin

## Prerequisites

The easiest way to start using these event types is to install the WordPress plugin [Noti - Activity Notifications](https://wordpress.org/plugins/noti-activity-notification).

## Configurations Overview

Event type configuration is based on JSON format and contains only a few key properties. Under the hood, it is a fork of [JSON Policy](https://jsonpolicy.github.io/) open-source project that I maintain. So let's take a simple example and digest it in greater detail.

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

The `Event` property declares what type of hook we are going to be listening to. It starts with either `filter:` or `action:` following by the name of the hook (in our example it is `pre_update_site_option_auto_update_themes`). WordPress core has hundreds of hooks you can "subscribe" to and most mature plugins or themes also introduce their own hooks.

> FYI! It is important to pay attention to the type of hook and correctly state it for the `Event` property. WordPress filters always return a value, while actions do not. That is why if you made a mistake instead of `filter:` used `action:`, you most likely will break some functionality.

The `Level` property is optional and can be used to declare the "level of event importance". For example, if a user modified some plugin's file, you probably want to consider this type of event as "Critical". However, if a draft post is modified, this may not be as important and can be labeled as "Notice". It is up to you and your business nature to define what is more or less important. This information is used only for your convenience to better work with a large list of events in the log.

The `RequiredVersion` property is also optional, however, can be a critical piece of information to let you know what is the minimum required version of software you need so your event type can function properly. Some hooks may not be available or have a different set of parameters in an earlier version.

The `Metadata` is a map of key/value pairs that declares what needs to be captured and stored in the database as metadata when an event is triggered. This information is stored in the `noti_eventmeta` database table. You have full freedom to capture as much information as needed and below I'll explain in more detail about the concept of markers (e.g. `${USER.ID}`, `${FUNC.wp_get_theme(HTTP_POST.asset).name}`).

The `Condition` is where the great power of these configurations comes from. Basically here you can define a set of conditions under which event should be captured. For example, you can define a custom event type that captures when ONLY users with subscriber roles posted a comment, or when any post in the "Science" category was modified. There is no limit to what you can do here. For more information about conditions, you can refer to my documentation for the [JSON Policy Conditions](https://jsonpolicy.github.io/overview/policy-condition.html).

The `MessageMarkdown` is optional, however, useful way to construct your own custom message that you'll see on the "Activity Log" page inside your WordPress dashboard. This message property is also used to compile the message that is sent as a notification to your email box. It heavily relies on the information that is captured when an event occurred, so if you need to be more verbose with your message, you probably would want to declare more key/value pairs in the `Metadata` property.

The `Notifications` contains the array of notification types that will be triggered to route captured events to a different destination(s). In the initial Noti version, to avoid performance issues, events are captured in the database first and then routed elsewhere with a WordPress cron job that occurs every minute. This behavior may be changed in later Noti versions if more near-to-realtime notifications will be needed. A bit more information about currently supported notification types you can find below.

There are also two additional supported properties `Aggregate` and `Listeners` that are more advanced topics and I'm explaining them in greater detail below.

## Configuration Markers

Marker (or some may think about it as a token) is a piece of configuration that is replaced with some value at runtime (at the time the event is triggered or notification is prepared to be sent). This functionality is based on the [JSON Policy Marker](https://jsonpolicy.github.io/overview/policy-marker.html) concept and you can learn a bit more about it by following the mentioned link.

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
* `${LISTENER.N.metaName}` get meta data from any specific listener (advanced topic I'm covering below).
* `${EVENT_META.metaName}` get captured event's meta data. Basically any captured properties defined in the `Metadata`.
* `${EVENT.propertyName}` get event's property. Basically any column from the `noti_events` database table (e.g. `site_id`, `counter`, `last_occurrence_at`, etc).
* `${EVENT_TYPE.propertyName}` get event types property. Under the hood, event type is just another custom post type that is stored in the `posts` database table. You can access any column with this marker (e.g. `post_modified`, `post_author`, `post_title`, etc.).
* `${EVENT_TYPE_META.metaName}` get meta data from event type. Basically any metadata value from `postmeta` database table.

## Notification Types

Noti uses an automated WordPress cron-job scheduler that runs every minute and routes aggregated list of "sealed" events to one or more destinations (for more information about the consolidation of the same event and when an event is considered "sealed", check the "Aggregate" section below). Currently, three distinct destinations are implemented and more will come as the plugin evolves.

> FYI! WordPress cron-job is actually is not something that triggers itself on regular basis and depends on how often the website is used. That is why there is no guarantee that newly captured events will be sent within a minute.

The default and global configurations for all supported notification types can be found on the "Activity Log -> Settings" page and they are as following:

```json
[
    {
        "Type": "email",
        "Status": "active",
        "Subject": "WP Activity Notifications",
        "BodyTemplate": "${WP_NETWORK_OPTION.noti-email-notification-tmpl}",
        "SendAsHTML": true
    },
    {
        "Type": "file",
        "Status": "inactive",
        "Filepath": "${CONST.ABSPATH}/../youlogfile.log",
        "MessageMarkdown": "[${EVENT.first_occurrence_at}] ${FUNC.ucfirst(EVENT_META.level)} (${EVENT.counter}): ${FUNC.get_userdata(EVENT_META.user_id).display_name} (ID: ${EVENT_META.user_id}) triggered the ${EVENT_TYPE.post_title} event"
    },
    {
        "Status": "inactive",
        "Type": "webhook",
        "Url": "https://yourdesiredurl",
        "Method": "POST",
        "Headers": [
            "Authorization: Bearer jkkj",
            "Content-Type: application/json"
        ],
        "Payload": {
            "event_name": "${EVENT_TYPE.post_title}",
            "user_id": "${EVENT_META.user_id}"
        }
    }
]
```

These configurations are global because they are inherited by every event type and you have the ability to override any global property within the event type's configuration.

## Aggregates

The aggregates are designed to solve the typical problem of capturing repetitive information over a certain period of time. For example, let's say you have a WordPress site that facilitates 100+ writers that continuously write new blogs or edit existing ones. Every time a writer selects to save a blog, a new event "Post Updated" is captured with some additional metadata. When the same writer selects to save a blog 10 times over the period of 30 minutes, your database will contain 10 very similar events with exactly the same metadata. That is 10x your database storage usage unnecessarily. On the same note, if you need to capture the event when the user failed to log in, you may end up with millions of "Login Failed Attempt" events if a password brute force attack was performed.

The `Aggregate` property gives you the ability to aggregate (or rather consolidate) events by a certain set of properties as well as time segment. Here is the example:

```json
{
    "Event": "action:save_post",
    "Aggregate": {
        "ByAttributes": [
            "${USER.ID}",
            "${ARGS.1.ID}"
        ],
        "ByTimeSegment": "1800s"
    }
}
```

This will create only one event record in the database when the same user saves the same post multiple times over the time segment of 30 minutes (1800 seconds). The `counter` column (in the `events` database table) will contain the total number of times the event occurred over a specified segment of time.

> Note! Please understand that under the time segment I mean a finite number of time units (e.g. seconds, days, months) starting since January 1st, 1970. For example, if `ByTimeSegment` equals `1i` (one minute), the same events that occurred on 2022-01-01 15:03:59 and then on 2022-01-01 15:04:01 (two seconds later) will be stored in a database as two separate records as they belong to two different time segments. However, the same event that occurred hundred times between 2022-01-01 15:03:00 and 2022-01-01 15:03:59 will result in only one database record with the `counter` column equaling 100.

The available time units are:

* `y` for year
* `m` for month
* `d` for day
* `h` for hour
* `i` for minute
* `s` for second

## Listeners

In some instances, subscribing to a very specific hook does not give us all the necessary details. The perfect example is the "Plugin Deleted" event when `deleted_plugin` action is triggered after a plugin is physically deleted. So there is no really a way to capture the name of the plugin that was just deleted or any additional details about it. That is why you can define one or more listeners for actions or filters that occur before the final hook and store in memory critical pieces of information.

```json
{
    "Event": "action:deleted_plugin",
    "Level": "critical",
    "Listeners": [
        {
            "Event": "action:delete_plugin",
            "Metadata": {
                "plugin": "${FUNC.get_plugin_data(ARGS.0).Name}"
            }
        }
    ],
    "Metadata": {
        "user_id": "${USER.ID}",
        "user_ip": "${USER.ip}",
        "plugin": "${LISTENER.0.plugin}"
    }
}
```

We know that the `delete_plugin` action is triggered before physical files are deleted, so it is a perfect place to gather all the necessary information about the about-to-be-deleted plugin and then persist it when `deleted_plugin` is invoked.

## Contributing To The Project
To contribute to the project, please follow these steps:

1. Fork this repository.
2. Create a branch: `git checkout -b <branch_name>`.
3. Make your changes and commit them: `git commit -m '<commit_message>'`
4. Push to the original branch: `git push origin <project_name>/<location>`
5. Create the pull request.

Alternatively, see the GitHub documentation on [creating a pull request](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/creating-a-pull-request).


## Contact

If you want to contact me you can reach me at <vasyl@vasyltech.com>.


## License

This project uses the following license: [<GNU General Public License>](https://www.gnu.org/licenses/#GPL).