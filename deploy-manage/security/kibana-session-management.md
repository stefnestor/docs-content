---
mapped_pages:
  - https://www.elastic.co/guide/en/kibana/current/xpack-security-session-management.html
applies_to:
  deployment:
    ess: ga
    ece: ga
    eck: ga
    self: ga
products:
  - id: kibana
---

# {{kib}} session management [xpack-security-session-management]

When you log in, {{kib}} creates a session that is used to authenticate subsequent requests to {{kib}}. A session consists of two components: an encrypted cookie that is stored in your browser, and an encrypted document in a dedicated {{es}} hidden index. By default, the name of that index is `.kibana_security_session_1`, where the prefix is derived from the primary `.kibana` index. If either of these components are missing, the session is no longer valid.

When your session expires, or you log out, {{kib}} will invalidate your cookie and remove session information from the index. {{kib}} also periodically invalidates and removes any expired sessions that weren’t explicitly invalidated.

To manage user sessions programmatically, {{kib}} exposes [session management APIs](https://www.elastic.co/docs/api/doc/kibana/group/endpoint-user-session). For details, check out [Session and cookie security settings](kibana://reference/configuration-reference/security-settings.md#security-session-and-cookie-settings).

## Session idle timeout [session-idle-timeout]

You can use `xpack.security.session.idleTimeout` to expire sessions after a period of inactivity. This and `xpack.security.session.lifespan` are both highly recommended. By default, sessions expire after 3 days of inactivity. To define another value for a sliding session expiration, set the  property in the [`kibana.yml`](/deploy-manage/stack-settings.md) configuration file. The idle timeout is formatted as a duration of `<count>[ms|s|m|h|d|w|M|Y]` (e.g. *20m*, *24h*, *7d*, *1w*). For example, set the idle timeout to expire sessions after 30 minutes of inactivity:

```yaml
xpack.security.session.idleTimeout: "30m"
```


## Session lifespan [session-lifespan]

You can use `xpack.security.session.lifespan` to configure the maximum session duration or "lifespan" — also known as the "absolute timeout". This and `xpack.security.session.idleTimeout` are both highly recommended. By default, a maximum session lifespan is 30 days. To define another lifespan, set the property in the [`kibana.yml`](/deploy-manage/stack-settings.md) configuration file. The lifespan is formatted as a duration of `<count>[ms|s|m|h|d|w|M|Y]` (e.g. *20m*, *24h*, *7d*, *1w*). For example, set the lifespan to expire sessions after 7 days:

```yaml
xpack.security.session.lifespan: "7d"
```


## Session cleanup interval [session-cleanup-interval]

::::{important}
If you disable session idle timeout and lifespan, then {{kib}} will not automatically remove session information from the index unless you explicitly log out. This might lead to an infinitely growing session index. As long as either idle timeout or lifespan is configured, {{kib}} sessions will be cleaned up even if you don’t explicitly log out.

::::


You can configure the interval at which {{kib}} tries to remove expired and invalid sessions from the session index. By default, this value is 1 hour and cannot be less than 10 seconds. To define another interval, set the `xpack.security.session.cleanupInterval` property in the [`kibana.yml`](/deploy-manage/stack-settings.md) configuration file. The interval is formatted as a duration of `<count>[ms|s|m|h|d|w|M|Y]` (e.g. *20m*, *24h*, *7d*, *1w*). For example, schedule the session index cleanup to perform once a day:

```yaml
xpack.security.session.cleanupInterval: "1d"
```


## Maximum number of concurrent sessions [session-max-sessions]

By default, there is no limit to the maximum number of concurrent sessions each user can have in {{kib}}. To add a limit, use the `xpack.security.session.сoncurrentSessions.maxSessions` configuration option. If set, the value of this option should be an integer between `1` and `1000`. When the limit is exceeded, the oldest session is automatically invalidated.

::::{note}
Due to the rate at which session information is refreshed, there might be a few seconds where the concurrent session limit is not enforced. This is something to consider for use cases where it is common to create multiple sessions simultaneously.
::::


```yaml
xpack.security.session.concurrentSessions:
  maxSessions: 3
```


