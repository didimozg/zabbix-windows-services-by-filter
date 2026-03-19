# Zabbix Windows Services by Filter

![Zabbix](https://img.shields.io/badge/Zabbix-7.4-red)
![Platform](https://img.shields.io/badge/Platform-Windows-lightgrey)
![Data source](https://img.shields.io/badge/Data%20source-Zabbix%20Agent%202-blue)
![License](https://img.shields.io/badge/License-MIT-green)
![Maintenance](https://img.shields.io/badge/Maintained-yes-brightgreen)

Zabbix template for monitoring **an arbitrary number of Windows services** using **service discovery**, **LLD filtering**, and **host-level macros**.

The template is designed to monitor only explicitly selected services.  
By default, it creates **no discovered services at all** until a host overrides the matching macro.

---

# Features

- discovers Windows services via `service.discovery`
- monitors **any number of services**
- uses host macros to control which services are included
- shows service state **as text**, not as numeric codes
- supports include and exclude filtering
- safe by default:
  - if no macro is defined on the host, **nothing is discovered**
- no external scripts required

---

# Architecture

```text
Windows service discovery
        ↓
LLD filter by host macro
        ↓
Item prototypes
        ↓
Trigger prototypes
```

The template discovers all Windows services, then filters them using host-level regex macros.

Only matched services are monitored.

---

# Repository Structure

```text
.
└─ template_windows_services_by_filter_disabled_by_default.yaml
```

---

# How It Works

## 1. Discovery

The template uses the built-in Zabbix agent key:

```text
service.discovery
```

This discovers Windows services on the host.

## 2. LLD filtering

The discovered list is filtered using the macro:

```text
{$WIN.SVC.MATCH}
```

By default, the template sets:

```text
{$WIN.SVC.MATCH}=^$
```

This regex matches only an empty string, so **no real service names match it**.

As a result:

- the template can be linked safely to many hosts
- if the macro is not overridden on a host, **no service items or triggers are created**

## 3. Optional exclusion filter

The template also supports an exclusion macro:

```text
{$WIN.SVC.NOT_MATCHES}
```

It can be used to remove specific services from the discovered set.

Example:

```text
{$WIN.SVC.NOT_MATCHES}=^RemoteRegistry$
```

---

# Supported Monitoring

For each matched service, the template creates:

- service state code
- service state as text
- startup type code
- startup type as text
- display name
- description

It also creates triggers for:

- service is not running
- service does not exist

---

# Macros

## Required host macro to enable monitoring

```text
{$WIN.SVC.MATCH}
```

### Default template value

```text
^$
```

This means:

- **default behavior = discover nothing**
- monitoring starts only after a host explicitly overrides this macro

---

## Optional exclusion macro

```text
{$WIN.SVC.NOT_MATCHES}
```

### Default template value

```text
^$
```

---

## Failure window macro

```text
{$WIN.SVC.FAIL.WINDOW}
```

### Default value

```text
5m
```

Used in trigger logic to avoid false positives during temporary service transitions.

---

# Examples

## Monitor a single service

```text
{$WIN.SVC.MATCH}=^Spooler$
```

## Monitor multiple services

```text
{$WIN.SVC.MATCH}=^(Spooler|W32Time|Dnscache|LanmanWorkstation)$
```

## Monitor all Zabbix-related services

```text
{$WIN.SVC.MATCH}=^Zabbix.*
```

## Monitor several services but exclude one

```text
{$WIN.SVC.MATCH}=^(Spooler|W32Time|RemoteRegistry)$
{$WIN.SVC.NOT_MATCHES}=^RemoteRegistry$
```

---

# Created Items

For each matched service, the template creates:

- `_INTERNAL_: Service {#SERVICE.NAME} state code`
- `Service {#SERVICE.NAME} ({#SERVICE.DISPLAYNAME}) state`
- `_INTERNAL_: Service {#SERVICE.NAME} startup code`
- `Service {#SERVICE.NAME} ({#SERVICE.DISPLAYNAME}) startup type`
- `Service {#SERVICE.NAME} ({#SERVICE.DISPLAYNAME}) display name`
- `Service {#SERVICE.NAME} ({#SERVICE.DISPLAYNAME}) description`

The internal numeric items are used for trigger evaluation, while user-facing items show readable text values.

---

# Human-Readable States

The template converts service state codes into readable text values such as:

- `Running`
- `Stopped`
- `Paused`
- `Start pending`
- `Stop pending`
- `Does not exist`

The same approach is used for startup type display.

---

# Triggers

For each matched service, the template creates the following triggers:

## Service is not running

Triggers if the service exists but is not in the `Running` state for the configured failure window.

## Service does not exist

Triggers if the service is not present on the host.

---

# Requirements

- **Zabbix 7.4**
- **Zabbix Agent 2**
- Windows host
- access to standard Windows service information via Zabbix agent

---

# Installation

## 1. Import the template

Import:

```text
template_windows_services_by_filter_disabled_by_default.yaml
```

via:

```text
Data collection → Templates → Import
```

---

## 2. Link the template to a host

You can safely link the template to a host without creating any service items immediately.

---

## 3. Enable monitoring for the host

Override the host macro:

```text
{$WIN.SVC.MATCH}
```

Example:

```text
{$WIN.SVC.MATCH}=^(Spooler|W32Time|Dnscache)$
```

Only after this, matching services will be discovered and monitored.

---

# Why This Design

This template is intentionally **disabled by default**.

This provides a clean and safe operational model:

- the same template can be linked to many hosts
- hosts are not cluttered with irrelevant service items
- monitoring is enabled only where explicitly needed
- service selection is controlled centrally via macros

This is especially useful in large environments where only certain hosts need monitoring for certain services.

---

# Troubleshooting

## 1. No services are discovered

### Cause

Most likely, `{$WIN.SVC.MATCH}` was not overridden on the host.

### Fix

Set a host macro such as:

```text
{$WIN.SVC.MATCH}=^Spooler$
```

---

## 2. Discovery creates too many services

### Cause

The match regex is too broad.

### Example of broad filter

```text
{$WIN.SVC.MATCH}=.*
```

### Fix

Use a stricter regex, for example:

```text
{$WIN.SVC.MATCH}=^(Spooler|W32Time)$
```

---

## 3. A service should be monitored but is missing

### Checks

- verify the actual Windows service name
- verify whether the service name matches the regex
- verify that it is not excluded by `{$WIN.SVC.NOT_MATCHES}`

---

## 4. Service state is shown as text, but triggers do not work as expected

### Cause

Triggers use the internal numeric state item, not the text item.

### Fix

Make sure the internal state code item exists and has values.

---

## 5. Service does not exist trigger fires unexpectedly

### Cause

The service name pattern may be incorrect.

### Fix

Check whether the real service name matches your regex exactly.

For example, use:

```text
^W32Time$
```

instead of a broader or incorrect pattern.
<img width="840" height="252" alt="изображение" src="https://github.com/user-attachments/assets/06bee86b-bb3c-4904-98f3-f4c7a4020fab" />

---

# Advantages

- no external scripts
- no per-service template cloning
- supports unlimited number of services
- safe default behavior
- easy to scale in large environments
- readable states in the UI

---

# Limitations

- Windows-only
- based on Windows service discovery
- relies on regex matching for service selection
- service names must be known or verified

---

# Author

This template was created for selective monitoring of Windows services in Zabbix using discovery and host-level filtering macros.
