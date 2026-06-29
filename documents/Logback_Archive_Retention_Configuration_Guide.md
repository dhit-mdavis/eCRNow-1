# Logback Archive Retention Configuration Guide

## Overview

The eCRNow application uses **Logback** (via Spring Boot's `logback-spring.xml`) to manage application logging. Log files are rolled over daily and archived as compressed `.gz` files. This guide walks through the current configuration and how to adjust the number of days that archived log files are retained.

---

## Current Configuration

The active logback configuration file is located at:

```
src/main/resources/logback-spring.xml
```

The application already uses a fully custom `RollingFileAppender` — it does **not** delegate to Spring Boot's default `file-appender.xml`. The current configuration keeps **30 days** of daily archives and runs a history cleanup on startup:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

  <!-- Pull logging.file.name from Spring Boot config (application.properties) -->
  <springProperty scope="context" name="LOG_FILE" source="logging.file.name"/>

  <!-- Fallback if logging.file.name is not set -->
  <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}/}spring.log}"/>

  <!-- Daily rolling file appender, keep 30 days -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_FILE}</file>
    <append>true</append>

    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <!-- Daily rollover — archive files are named with the date they were rolled -->
      <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.gz</fileNamePattern>

      <!-- Keep the last 30 days of archive files -->
      <maxHistory>30</maxHistory>

      <!-- Delete archives older than maxHistory on application startup -->
      <cleanHistoryOnStart>true</cleanHistoryOnStart>
    </rollingPolicy>

    <encoder>
      <pattern>${FILE_LOG_PATTERN}</pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="FILE"/>
  </root>

</configuration>
```

---

## How to Adjust the Number of Days Archives Are Kept

The retention period is controlled by a single element inside the `<rollingPolicy>` block:

```xml
<maxHistory>30</maxHistory>
```

**To change the retention period**, open `src/main/resources/logback-spring.xml` and update the integer value:

```xml
<!-- Keep 7 days (one week) -->
<maxHistory>7</maxHistory>

<!-- Keep 30 days (one month) — current setting -->
<maxHistory>30</maxHistory>

<!-- Keep 90 days (three months) -->
<maxHistory>90</maxHistory>

<!-- Keep 365 days (one year) -->
<maxHistory>365</maxHistory>
```

At each daily rollover (midnight), Logback automatically deletes archives older than `maxHistory` days. Because `<cleanHistoryOnStart>true</cleanHistoryOnStart>` is also enabled, stale archives are also purged whenever the application starts up — useful after a period of downtime.

---

## Key Configuration Properties

| Property | XML Element | Description | Current Value |
|---|---|---|---|
| Days to retain archives | `<maxHistory>` | Number of daily archive files to keep; older files are automatically deleted | `30` |
| Startup cleanup | `<cleanHistoryOnStart>` | Purges archives older than `maxHistory` when the application starts | `true` |
| Archive filename pattern | `<fileNamePattern>` | Naming convention and compression format for daily archives | `${LOG_FILE}.%d{yyyy-MM-dd}.gz` |
| Active log file path | `<file>` | Path to the current log file being written | Resolved from `logging.file.name` in `application.properties` |
| Append mode | `<append>` | Appends to the existing log file rather than overwriting on startup | `true` |

### `<maxHistory>` Details

- Value is a plain **integer** representing **calendar days**.
- Logback deletes the oldest archive files once the count exceeds `maxHistory`.
- Common reference values:

  | Value | Retention Period |
  |---|---|
  | `7` | One week |
  | `14` | Two weeks |
  | `30` | One month *(current)* |
  | `90` | Three months |
  | `365` | One year |

### `<cleanHistoryOnStart>` Details

- When set to `true`, Logback scans the archive directory at startup and removes any files that fall outside the `maxHistory` window.
- This is particularly useful if the application was down for an extended period and missed one or more nightly cleanup cycles.
- Set to `false` if you prefer cleanup to happen only at rollover time (midnight).

### Adding a Total Size Cap (Optional)

If disk space is a concern, you can also add a `<totalSizeCap>` element to set an upper bound on the combined size of all archive files. Archives are pruned oldest-first if the cap is exceeded, even if `maxHistory` has not been reached:

```xml
<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
  <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.gz</fileNamePattern>
  <maxHistory>30</maxHistory>
  <cleanHistoryOnStart>true</cleanHistoryOnStart>

  <!-- Remove this element to disable the cap -->
  <totalSizeCap>1GB</totalSizeCap>
</rollingPolicy>
```

Accepted size suffixes: `KB`, `MB`, `GB`. Requires Logback **1.1.8 or later** (included in Spring Boot 2.x and 3.x).

---

## Controlling the Log File Path

The log file location is resolved using `<springProperty>` to read `logging.file.name` directly from `application.properties`, with a fallback chain if that property is not set:

| Source | Example |
|---|---|
| `logging.file.name` in `application.properties` *(primary)* | `/opt/ecrnow/logs/ecrnow-app.log` |
| `LOG_PATH` environment variable | `/opt/ecrnow/logs/spring.log` |
| `LOG_TEMP` environment variable | `<LOG_TEMP>/spring.log` |
| `java.io.tmpdir` JVM property | `/tmp/spring.log` |

**Recommended — set the log file path in `application.properties`:**

```properties
# Sets the full path and filename; logback-spring.xml reads this via <springProperty>
logging.file.name=/opt/ecrnow/logs/ecrnow-app.log
```

With this set, daily archives will be created as:

```
/opt/ecrnow/logs/ecrnow-app.log.2025-06-28.gz
/opt/ecrnow/logs/ecrnow-app.log.2025-06-29.gz
/opt/ecrnow/logs/ecrnow-app.log.2025-06-30.gz
```

Files older than `<maxHistory>` days are automatically removed.

---

## Applying the Change

1. Open `src/main/resources/logback-spring.xml`.
2. Update `<maxHistory>` to the desired number of days.
3. Optionally add `<totalSizeCap>` if disk usage needs to be bounded.
4. Rebuild and redeploy the application. Logback reads the configuration at startup — no additional reload step is required beyond the normal deployment process.

> **Docker deployments:** Ensure the log directory is mounted as a named volume so archive files persist across container restarts. Without a volume, all archived logs are lost when the container is recreated.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Old archives not being deleted | `maxHistory` not set or element is misplaced | Confirm `<maxHistory>` is nested inside `<rollingPolicy>`, not directly inside `<appender>` |
| Archives not cleaned up after downtime | `cleanHistoryOnStart` is `false` or missing | Set `<cleanHistoryOnStart>true</cleanHistoryOnStart>` inside `<rollingPolicy>` |
| All logs going to `/tmp` | `logging.file.name` not set in `application.properties` | Add `logging.file.name=/your/path/ecrnow-app.log` to `application.properties` |
| `totalSizeCap` has no effect | Logback version too old | Confirm Spring Boot version is 2.x or 3.x (Logback ≥ 1.1.8) |
| Configuration changes not applied | Cached build artifact | Clean and rebuild: `mvn clean package` |

---

## References

- [Logback RollingFileAppender Documentation](https://logback.qos.ch/manual/appenders.html#RollingFileAppender)
- [Spring Boot Logging Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)
- eCRNow active logback config: [`src/main/resources/logback-spring.xml`](../src/main/resources/logback-spring.xml)
