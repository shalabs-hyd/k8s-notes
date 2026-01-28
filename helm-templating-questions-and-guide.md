# Helm Templating - Complete Q&A Study Guide

> **Master Helm templating with practical examples, interview questions, and real-world scenarios**

---

## Table of Contents

1. [Helm Templating Basics](#helm-templating-basics)
2. [Built-in Objects](#built-in-objects)
3. [Functions and Pipelines](#functions-and-pipelines)
4. [Control Structures](#control-structures)
5. [Named Templates](#named-templates)
6. [Advanced Templating](#advanced-templating)
7. [Common Interview Questions](#common-interview-questions)
8. [Practical Examples](#practical-examples)
9. [Troubleshooting](#troubleshooting)
10. [Best Practices](#best-practices)

---

## Helm Templating Basics

### Q1: What is Helm templating and why is it used?

**Answer:**

Helm uses Go templating engine to generate Kubernetes manifest files dynamically. Templates allow you to:

- ✅ Parameterize configurations
- ✅ Reuse common patterns
- ✅ Generate multiple similar resources
- ✅ Make charts customizable
- ✅ Avoid hardcoding values

**Example:**
```yaml
# Instead of hardcoding:
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  
# Use template:
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-nginx
```

---

### Q2: What is the difference between `{{ }}` and `{{- -}}`?

**Answer:**

- `{{ }}` - Standard template delimiter, preserves whitespace
- `{{- }}` - Trim whitespace before
- `{{ -}}` - Trim whitespace after
- `{{- -}}` - Trim whitespace both sides

**Example:**
```yaml
# Without trimming
data:
  key1: {{ .Values.data1 }}
  key2: value

# With trimming (removes blank lines)
data:
  {{- if .Values.data1 }}
  key1: {{ .Values.data1 }}
  {{- end }}
  key2: value
```

**Output comparison:**
```yaml
# Without trimming:
data:

  key1: value1
  
  key2: value

# With trimming:
data:
  key1: value1
  key2: value
```

---

### Q3: How do you comment in Helm templates?

**Answer:**

**Template comments** (not rendered):
```yaml
{{/* This is a template comment */}}
{{/* 
Multi-line
template comment 
*/}}
```

**YAML comments** (visible in rendered output):
```yaml
# This is a YAML comment
apiVersion: v1
kind: ConfigMap
```

**When to use each:**
- Template comments: Document template logic
- YAML comments: Help users understand the generated YAML

---

## Built-in Objects

### Q4: What are the main built-in objects in Helm?

**Answer:**

| Object | Description | Example |
|--------|-------------|---------|
| `.Release` | Release information | `.Release.Name`, `.Release.Namespace` |
| `.Values` | Values from values.yaml | `.Values.image.repository` |
| `.Chart` | Chart.yaml contents | `.Chart.Name`, `.Chart.Version` |
| `.Files` | Access non-template files | `.Files.Get "config.txt"` |
| `.Capabilities` | Cluster capabilities | `.Capabilities.KubeVersion` |
| `.Template` | Current template info | `.Template.Name` |

**Practical Example:**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**values.yaml:**
```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.21"
```

---

### Q5: How do you access nested values in Values object?

**Answer:**

**values.yaml:**
```yaml
database:
  mysql:
    host: localhost
    port: 3306
    credentials:
      username: admin
      password: secret
```

**Template access:**
```yaml
# Method 1: Dot notation
host: {{ .Values.database.mysql.host }}

# Method 2: index function (useful for dynamic keys)
host: {{ index .Values "database" "mysql" "host" }}

# Accessing deeply nested
username: {{ .Values.database.mysql.credentials.username }}
username: {{ index .Values "database" "mysql" "credentials" "username" }}
```

---

### Q6: What is the `.Release` object and what properties does it have?

**Answer:**

The `.Release` object contains information about the current release:

| Property | Description | Example |
|----------|-------------|---------|
| `.Release.Name` | Release name | `my-release` |
| `.Release.Namespace` | Release namespace | `production` |
| `.Release.IsUpgrade` | Boolean if upgrade | `true` or `false` |
| `.Release.IsInstall` | Boolean if new install | `true` or `false` |
| `.Release.Revision` | Revision number | `1`, `2`, `3`... |
| `.Release.Service` | Service conducting release | `Helm` |

**Example Usage:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  namespace: {{ .Release.Namespace }}
  labels:
    release: {{ .Release.Name }}
    revision: {{ .Release.Revision | quote }}
data:
  {{- if .Release.IsInstall }}
  message: "Fresh installation!"
  {{- else if .Release.IsUpgrade }}
  message: "Upgrading existing release"
  {{- end }}
```

---

## Functions and Pipelines

### Q7: What are the most commonly used Helm functions?

**Answer:**

#### String Functions
```yaml
# quote - Add quotes
name: {{ .Values.name | quote }}
# Output: name: "myapp"

# upper - Uppercase
env: {{ .Values.environment | upper }}
# Output: env: PRODUCTION

# lower - Lowercase  
name: {{ .Values.appName | lower }}
# Output: name: myapp

# trim - Remove whitespace
cleaned: {{ .Values.data | trim }}

# trunc - Truncate string
short: {{ .Release.Name | trunc 6 }}
# Output: short: my-rel
```

#### Default Values
```yaml
# default - Provide fallback
replicas: {{ .Values.replicaCount | default 3 }}

# Required - Fail if missing
apiKey: {{ required "API key is required!" .Values.apiKey }}
```

#### Type Conversion
```yaml
# toJson
config: {{ .Values.config | toJson }}

# toYaml (very common)
labels:
{{ .Values.labels | toYaml | indent 2 }}

# toString
version: {{ .Chart.Version | toString }}
```

---

### Q8: What are pipelines and how do they work?

**Answer:**

Pipelines allow chaining multiple functions together using the `|` operator.

**Syntax:**
```
{{ value | function1 | function2 | function3 }}
```

**Examples:**

**Simple pipeline:**
```yaml
# Chain: trim -> lower -> quote
name: {{ .Values.appName | trim | lower | quote }}

# Input: " MyApp "
# After trim: "MyApp"
# After lower: "myapp"
# After quote: "myapp"
```

**Complex pipeline:**
```yaml
# Generate unique name
name: {{ .Release.Name | trunc 20 | trimSuffix "-" | lower }}

# Add default, then quote
timeout: {{ .Values.timeout | default "30s" | quote }}

# Convert to YAML and indent
{{- .Values.extraLabels | toYaml | nindent 2 }}
```

---

### Q9: How do you use the `default` function?

**Answer:**

The `default` function provides a fallback value when a variable is empty or undefined.

**Syntax:**
```yaml
{{ default "fallback-value" .Values.someValue }}
# Or with pipeline:
{{ .Values.someValue | default "fallback-value" }}
```

**Examples:**

```yaml
# values.yaml
replicaCount: 3
# environment not defined

# template.yaml
replicas: {{ .Values.replicaCount | default 1 }}
# Output: replicas: 3 (uses actual value)

environment: {{ .Values.environment | default "development" }}
# Output: environment: development (uses default)

# Complex default
image: {{ .Values.image.repository | default "nginx" }}:{{ .Values.image.tag | default "latest" }}
# Output: image: nginx:latest
```

**Important:** Empty string `""`, `false`, and `0` are NOT considered empty by default!

```yaml
# This WON'T use default
enabled: {{ .Values.feature.enabled | default true }}
# If .Values.feature.enabled is false, output is: false

# Use this instead for boolean checks:
{{- if not (hasKey .Values.feature "enabled") }}
enabled: true
{{- else }}
enabled: {{ .Values.feature.enabled }}
{{- end }}
```

---

### Q10: What is the difference between `quote` and `squote`?

**Answer:**

```yaml
# quote - Double quotes
name: {{ .Values.name | quote }}
# Output: "myapp"

# squote - Single quotes
name: {{ .Values.name | squote }}
# Output: 'myapp'

# Practical difference:
command: {{ .Values.command | quote }}
# Output with special chars: "echo \"hello\""

command: {{ .Values.command | squote }}
# Output with special chars: 'echo "hello"'
```

**When to use:**
- `quote`: Default choice, YAML standard
- `squote`: When value contains double quotes

---

### Q11: How do you use the `toYaml` and `toJson` functions?

**Answer:**

**toYaml** - Convert data structure to YAML:

```yaml
# values.yaml
labels:
  app: myapp
  environment: production
  team: platform

# template.yaml
metadata:
  labels:
{{ .Values.labels | toYaml | indent 4 }}
```

**Output:**
```yaml
metadata:
  labels:
    app: myapp
    environment: production
    team: platform
```

**toJson** - Convert to JSON:

```yaml
# values.yaml
config:
  database:
    host: localhost
    port: 5432

# template.yaml
data:
  config.json: |
{{ .Values.config | toJson | indent 4 }}
```

**Output:**
```yaml
data:
  config.json: |
    {
      "database": {
        "host": "localhost",
        "port": 5432
      }
    }
```

**Common issue and solution:**
```yaml
# ❌ Wrong - missing indent
labels:
{{ .Values.labels | toYaml }}

# ✅ Correct - with indent
labels:
{{ .Values.labels | toYaml | indent 2 }}

# ✅ Even better - with nindent (adds newline + indent)
labels:
{{- .Values.labels | toYaml | nindent 2 }}
```

---

### Q12: What are `indent` and `nindent` functions?

**Answer:**

**indent** - Add specified number of spaces:
```yaml
labels:
{{ .Values.labels | toYaml | indent 2 }}
# Adds 2 spaces before each line
```

**nindent** - Newline + indent:
```yaml
labels:
{{- .Values.labels | toYaml | nindent 2 }}
# Adds newline, then indents
```

**Practical comparison:**

```yaml
# Using indent
metadata:
  labels:
{{ .Values.labels | toYaml | indent 4 }}

# Output:
metadata:
  labels:
    app: myapp       # Extra blank line above
    env: prod

# Using nindent (better!)
metadata:
  labels:
{{- .Values.labels | toYaml | nindent 4 }}

# Output:
metadata:
  labels:
    app: myapp       # No blank line!
    env: prod
```

**Rule of thumb:** Use `nindent` with `{{-` for cleaner output!

---

## Control Structures

### Q13: How do you use if/else conditions in Helm?

**Answer:**

**Basic if:**
```yaml
{{- if .Values.enabled }}
enabled: true
{{- end }}
```

**if/else:**
```yaml
{{- if .Values.production }}
environment: production
{{- else }}
environment: development
{{- end }}
```

**if/else if/else:**
```yaml
{{- if eq .Values.environment "production" }}
replicas: 5
{{- else if eq .Values.environment "staging" }}
replicas: 3
{{- else }}
replicas: 1
{{- end }}
```

**Practical example:**
```yaml
# values.yaml
ingress:
  enabled: true
  tls:
    enabled: true
    secretName: my-tls-secret

# template
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  {{- if .Values.ingress.tls.enabled }}
  tls:
  - secretName: {{ .Values.ingress.tls.secretName }}
  {{- end }}
  rules:
  - host: {{ .Values.ingress.host }}
{{- end }}
```

---

### Q14: What logical operators are available in Helm templates?

**Answer:**

| Operator | Function | Example |
|----------|----------|---------|
| `eq` | Equal | `{{ if eq .Values.env "prod" }}` |
| `ne` | Not equal | `{{ if ne .Values.count 0 }}` |
| `lt` | Less than | `{{ if lt .Values.count 10 }}` |
| `le` | Less or equal | `{{ if le .Values.count 10 }}` |
| `gt` | Greater than | `{{ if gt .Values.count 5 }}` |
| `ge` | Greater or equal | `{{ if ge .Values.count 5 }}` |
| `and` | Logical AND | `{{ if and .Values.a .Values.b }}` |
| `or` | Logical OR | `{{ if or .Values.a .Values.b }}` |
| `not` | Logical NOT | `{{ if not .Values.disabled }}` |

**Examples:**

```yaml
# Equality
{{- if eq .Values.database.type "mysql" }}
port: 3306
{{- end }}

# Multiple conditions with AND
{{- if and .Values.ingress.enabled .Values.ingress.tls.enabled }}
tls:
  - secretName: {{ .Values.ingress.tls.secretName }}
{{- end }}

# Multiple conditions with OR
{{- if or (eq .Values.env "prod") (eq .Values.env "staging") }}
monitoring: enabled
{{- end }}

# Negation
{{- if not .Values.debug }}
logLevel: info
{{- else }}
logLevel: debug
{{- end }}

# Complex condition
{{- if and (eq .Values.env "production") (gt .Values.replicas 1) }}
podDisruptionBudget:
  enabled: true
{{- end }}
```

---

### Q15: How do you use range loops in Helm?

**Answer:**

**Iterate over list:**

```yaml
# values.yaml
environments:
  - dev
  - staging
  - prod

# template
{{- range .Values.environments }}
- name: {{ . }}
{{- end }}

# Output:
- name: dev
- name: staging
- name: prod
```

**Iterate over map:**

```yaml
# values.yaml
labels:
  app: myapp
  team: platform
  env: production

# template
metadata:
  labels:
{{- range $key, $value := .Values.labels }}
    {{ $key }}: {{ $value }}
{{- end }}

# Output:
metadata:
  labels:
    app: myapp
    team: platform
    env: production
```

**Complex example with index:**

```yaml
# values.yaml
databases:
  - name: mysql
    port: 3306
  - name: postgres
    port: 5432
  - name: mongo
    port: 27017

# template
services:
{{- range $index, $db := .Values.databases }}
  - name: {{ $db.name }}-{{ $index }}
    port: {{ $db.port }}
{{- end }}

# Output:
services:
  - name: mysql-0
    port: 3306
  - name: postgres-1
    port: 5432
  - name: mongo-2
    port: 27017
```

---

### Q16: What is the scope issue with range and how do you solve it?

**Answer:**

Inside `range`, `.` refers to the current item, NOT the root context!

**Problem:**
```yaml
# values.yaml
appName: myapp
databases:
  - mysql
  - postgres

# ❌ This FAILS
{{- range .Values.databases }}
name: {{ .Values.appName }}-{{ . }}  # ERROR: .Values doesn't exist here!
{{- end }}
```

**Solution 1: Use `$` for root context**
```yaml
{{- range .Values.databases }}
name: {{ $.Values.appName }}-{{ . }}  # ✅ $ refers to root
{{- end }}

# Output:
name: myapp-mysql
name: myapp-postgres
```

**Solution 2: Save context before range**
```yaml
{{- $appName := .Values.appName }}
{{- range .Values.databases }}
name: {{ $appName }}-{{ . }}  # ✅ Use saved variable
{{- end }}
```

**Solution 3: Use `with` to restore context**
```yaml
{{- range .Values.databases }}
{{- with $ }}  # Restore root context
name: {{ .Values.appName }}-{{ . }}
{{- end }}
{{- end }}
```

**Best practice:** Always use `$` for root context in loops!

---

### Q17: How do you use the `with` block?

**Answer:**

`with` changes the scope (`.`) to a specific value and only renders if value exists.

**Basic usage:**
```yaml
# values.yaml
database:
  mysql:
    host: localhost
    port: 3306

# Without with
host: {{ .Values.database.mysql.host }}
port: {{ .Values.database.mysql.port }}

# With 'with' (cleaner!)
{{- with .Values.database.mysql }}
host: {{ .host }}
port: {{ .port }}
{{- end }}
```

**Conditional rendering:**
```yaml
# Only render if .Values.ingress exists
{{- with .Values.ingress }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ $.Release.Name }}-ingress  # Note: use $ for root!
spec:
  rules:
  - host: {{ .host }}  # .host from .Values.ingress
{{- end }}
```

**with/else:**
```yaml
{{- with .Values.customConfig }}
# Use custom config
config: {{ .data }}
{{- else }}
# Use default config
config: "default"
{{- end }}
```

---

## Named Templates

### Q18: What are named templates and how do you create them?

**Answer:**

Named templates (also called "partials") are reusable template snippets defined with `define`.

**Define a named template:**
```yaml
# templates/_helpers.tpl
{{- define "mychart.labels" -}}
app: {{ .Chart.Name }}
version: {{ .Chart.Version }}
release: {{ .Release.Name }}
{{- end -}}
```

**Use the named template:**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
{{ include "mychart.labels" . | indent 4 }}
```

**Convention:**
- File name: `_helpers.tpl` (underscore = not rendered as manifest)
- Template name: `chartname.templatename`
- Always pass context: `{{ include "mychart.labels" . }}`

---

### Q19: What's the difference between `template` and `include`?

**Answer:**

| Feature | `template` | `include` |
|---------|-----------|-----------|
| **Returns** | Renders directly | Returns string |
| **Pipelines** | ❌ Cannot pipe | ✅ Can pipe to functions |
| **Indentation** | ❌ Manual | ✅ Use with `indent` |
| **Recommended** | ❌ Avoid | ✅ Prefer this |

**Example showing the difference:**

```yaml
{{- define "mychart.labels" -}}
app: {{ .Chart.Name }}
version: {{ .Chart.Version }}
{{- end -}}

# ❌ Using template (can't indent properly)
metadata:
  labels:
{{ template "mychart.labels" . }}

# Output (WRONG indentation):
metadata:
  labels:
app: myapp
version: 0.1.0

# ✅ Using include (can pipe to indent)
metadata:
  labels:
{{ include "mychart.labels" . | indent 4 }}

# Output (CORRECT indentation):
metadata:
  labels:
    app: myapp
    version: 0.1.0
```

**Rule:** Always use `include`, never `template`!

---

### Q20: How do you pass data to named templates?

**Answer:**

**Method 1: Pass entire context**
```yaml
{{- define "mychart.name" -}}
{{ .Chart.Name }}-{{ .Release.Name }}
{{- end -}}

# Usage:
name: {{ include "mychart.name" . }}
```

**Method 2: Pass specific values using dict**
```yaml
{{- define "mychart.image" -}}
{{ .repository }}:{{ .tag }}
{{- end -}}

# Usage:
image: {{ include "mychart.image" (dict "repository" .Values.image.repo "tag" .Values.image.tag) }}
```

**Method 3: Pass list of values**
```yaml
{{- define "mychart.url" -}}
{{ index . 0 }}://{{ index . 1 }}:{{ index . 2 }}
{{- end -}}

# Usage:
url: {{ include "mychart.url" (list "https" "example.com" "443") }}
```

**Practical example:**
```yaml
# _helpers.tpl
{{- define "mychart.resourceName" -}}
{{- $name := .name -}}
{{- $release := .release -}}
{{- printf "%s-%s" $release $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

# deployment.yaml
metadata:
  name: {{ include "mychart.resourceName" (dict "name" "web" "release" .Release.Name) }}
```

---

## Advanced Templating

### Q21: How do you use the `lookup` function?

**Answer:**

`lookup` queries the Kubernetes API during rendering to get existing resources.

**Syntax:**
```yaml
{{ lookup "apiVersion" "kind" "namespace" "name" }}
```

**Examples:**

**Get existing secret:**
```yaml
{{- $secret := lookup "v1" "Secret" .Release.Namespace "my-secret" }}
{{- if $secret }}
# Secret exists, use existing data
data:
  password: {{ $secret.data.password }}
{{- else }}
# Generate new password
data:
  password: {{ randAlphaNum 16 | b64enc }}
{{- end }}
```

**Get ConfigMap:**
```yaml
{{- $cm := lookup "v1" "ConfigMap" "default" "my-config" }}
{{- if $cm }}
existingValue: {{ $cm.data.key1 }}
{{- end }}
```

**List all resources:**
```yaml
{{- $services := lookup "v1" "Service" "" "" }}
{{- range $services.items }}
- {{ .metadata.name }}
{{- end }}
```

**⚠️ Important Notes:**
- Only works with `--dry-run=server` or actual install/upgrade
- Requires cluster connectivity
- Returns empty during `helm template` (client-side rendering)

---

### Q22: What is the `tpl` function and when do you use it?

**Answer:**

`tpl` evaluates a string as a template, allowing dynamic template rendering.

**Use case 1: Render values.yaml as templates**

```yaml
# values.yaml
message: "Hello from {{ .Release.Name }}"
greeting: "Welcome to {{ .Values.environment }}"
environment: production

# template.yaml
data:
  msg: {{ tpl .Values.message . }}
  greet: {{ tpl .Values.greeting . }}

# Output:
data:
  msg: Hello from my-release
  greet: Welcome to production
```

**Use case 2: Dynamic configuration**

```yaml
# values.yaml
config: |
  server:
    name: {{ .Release.Name }}-server
    port: {{ .Values.port }}
port: 8080

# template.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  app.conf: |
{{ tpl .Values.config . | indent 4 }}

# Output:
data:
  app.conf: |
    server:
      name: my-release-server
      port: 8080
```

**Use case 3: Conditional templates in values**

```yaml
# values.yaml
dev:
  url: "{{ .Values.baseUrl }}/dev"
prod:
  url: "{{ .Values.baseUrl }}/prod"
baseUrl: "https://api.example.com"
environment: prod

# template.yaml
apiVersion: v1
kind: ConfigMap
data:
  {{- if eq .Values.environment "dev" }}
  api-url: {{ tpl .Values.dev.url . }}
  {{- else }}
  api-url: {{ tpl .Values.prod.url . }}
  {{- end }}
```

---

### Q23: How do you read files in Helm templates?

**Answer:**

Use `.Files` object to access files in the chart directory (except templates/).

**File structure:**
```
mychart/
├── Chart.yaml
├── values.yaml
├── templates/
│   └── configmap.yaml
└── config/
    ├── app.conf
    └── database.yml
```

**Method 1: `.Files.Get`**
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  app.conf: |
{{ .Files.Get "config/app.conf" | indent 4 }}
```

**Method 2: `.Files.GetBytes` (for binary)**
```yaml
data:
  image.png: {{ .Files.GetBytes "files/image.png" | b64enc }}
```

**Method 3: `.Files.Glob` (pattern matching)**
```yaml
# Include all .conf files
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configs
data:
{{- range $path, $content := .Files.Glob "config/*.conf" }}
  {{ base $path }}: |
{{ $content | indent 4 }}
{{- end }}
```

**Method 4: `.Files.AsConfig`**
```yaml
# Automatically format all files as ConfigMap data
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
{{ (.Files.Glob "config/*").AsConfig | indent 2 }}
```

**Method 5: `.Files.AsSecrets` (base64 encode)**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secrets
data:
{{ (.Files.Glob "secrets/*").AsSecrets | indent 2 }}
```

---

### Q24: How do you use the `required` function?

**Answer:**

`required` fails the rendering if a value is not provided, with a custom error message.

**Basic usage:**
```yaml
apiKey: {{ required "API key is required!" .Values.apiKey }}
```

**If apiKey is not in values.yaml:**
```bash
$ helm install myapp ./chart
Error: execution error at (chart/templates/deployment.yaml:10:12): 
API key is required!
```

**Multiple required fields:**
```yaml
database:
  host: {{ required "Database host is required" .Values.database.host }}
  user: {{ required "Database user is required" .Values.database.user }}
  password: {{ required "Database password is required" .Values.database.password }}
```

**With default (required only in production):**
```yaml
{{- if eq .Values.environment "production" }}
  sslCert: {{ required "SSL certificate required in production!" .Values.ssl.cert }}
{{- else }}
  sslCert: {{ .Values.ssl.cert | default "self-signed" }}
{{- end }}
```

**In named templates:**
```yaml
{{- define "mychart.validateConfig" -}}
{{- required "App name is required" .appName }}
{{- required "Environment is required" .environment }}
{{- end -}}

# Usage:
{{ include "mychart.validateConfig" (dict "appName" .Values.appName "environment" .Values.env) }}
```

---

### Q25: How do you generate random values in Helm?

**Answer:**

**Random functions:**

```yaml
# Random alphanumeric string
password: {{ randAlphaNum 16 }}
# Output: password: a3B5xR8kL2mN9qW1

# Random alpha string (letters only)
token: {{ randAlpha 20 }}
# Output: token: abcdefghijklmnopqrst

# Random numeric string
pin: {{ randNumeric 6 }}
# Output: pin: 482937

# Random ASCII
key: {{ randAscii 32 }}
# Output: key: ~!@#$%^&*()_+{}|:"<>?

# UUID
id: {{ uuidv4 }}
# Output: id: 550e8400-e29b-41d4-a716-446655440000
```

**Practical example - Generate secrets:**

```yaml
{{- $secret := lookup "v1" "Secret" .Release.Namespace "my-secret" }}
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  {{- if $secret }}
  # Preserve existing password on upgrade
  password: {{ $secret.data.password }}
  {{- else }}
  # Generate new password on install
  password: {{ randAlphaNum 24 | b64enc }}
  {{- end }}
```

**⚠️ Important:** Random functions generate NEW values on each `helm upgrade`!
Use `lookup` to preserve existing values.

---

### Q26: How do you encode/decode base64?

**Answer:**

**Encode to base64:**
```yaml
# b64enc
password: {{ .Values.password | b64enc }}
# Input: "mysecret"
# Output: password: bXlzZWNyZXQ=
```

**Decode from base64:**
```yaml
# b64dec
decoded: {{ .Values.encodedValue | b64dec }}
# Input: "bXlzZWNyZXQ="
# Output: decoded: mysecret
```

**Practical example - Secrets:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  # All values must be base64 encoded
  username: {{ .Values.database.username | b64enc }}
  password: {{ .Values.database.password | b64enc }}
  {{- if .Values.certificate }}
  cert.pem: {{ .Files.Get "certs/cert.pem" | b64enc }}
  {{- end }}
stringData:
  # stringData allows plain text (auto-encoded by K8s)
  config.json: |
{{ .Values.config | toJson | indent 4 }}
```

---

## Common Interview Questions

### Q27: How would you create a generic ConfigMap template that works for multiple applications?

**Answer:**

```yaml
# _helpers.tpl
{{- define "common.configmap" -}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "common.fullname" . }}
  labels:
{{ include "common.labels" . | indent 4 }}
data:
{{- range $key, $value := .Values.configData }}
  {{ $key }}: {{ $value | quote }}
{{- end }}
{{- end -}}

{{- define "common.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}

{{- define "common.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.Version }}
{{- end -}}

# Usage in values.yaml
configData:
  DATABASE_URL: "postgresql://localhost:5432/mydb"
  LOG_LEVEL: "info"
  FEATURE_FLAG_X: "true"

# Usage in template
{{ include "common.configmap" . }}
```

---

### Q28: How do you handle multiple environments (dev, staging, prod) in Helm?

**Answer:**

**Method 1: Separate values files**

```bash
# values-dev.yaml
replicaCount: 1
resources:
  limits:
    cpu: 100m
    memory: 128Mi
ingress:
  host: dev.example.com

# values-prod.yaml
replicaCount: 3
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
ingress:
  host: example.com

# Install
helm install myapp ./chart -f values-dev.yaml
helm install myapp ./chart -f values-prod.yaml
```

**Method 2: Environment key in values**

```yaml
# values.yaml
environments:
  dev:
    replicas: 1
    host: dev.example.com
  staging:
    replicas: 2
    host: staging.example.com
  prod:
    replicas: 5
    host: example.com

# template.yaml
{{- $env := index .Values.environments .Values.environment }}
spec:
  replicas: {{ $env.replicas }}
---
spec:
  rules:
  - host: {{ $env.host }}
```

**Method 3: Conditional logic**

```yaml
{{- if eq .Values.environment "production" }}
spec:
  replicas: 5
  resources:
    limits:
      cpu: 2000m
      memory: 4Gi
{{- else if eq .Values.environment "staging" }}
spec:
  replicas: 2
  resources:
    limits:
      cpu: 500m
      memory: 1Gi
{{- else }}
spec:
  replicas: 1
  resources:
    limits:
      cpu: 100m
      memory: 256Mi
{{- end }}
```

---

### Q29: How would you implement blue-green deployment using Helm?

**Answer:**

```yaml
# values.yaml
deployment:
  color: blue  # or green
  blue:
    image:
      tag: "v1.0"
    replicas: 3
  green:
    image:
      tag: "v2.0"
    replicas: 3

# deployment.yaml
{{- $color := .Values.deployment.color }}
{{- $config := index .Values.deployment $color }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ $color }}
  labels:
    app: {{ .Chart.Name }}
    color: {{ $color }}
spec:
  replicas: {{ $config.replicas }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
      color: {{ $color }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        color: {{ $color }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ $config.image.tag }}"

# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Chart.Name }}
    color: {{ .Values.deployment.color }}  # Points to active color
  ports:
  - port: 80

# Switch traffic:
# helm upgrade myapp ./chart --set deployment.color=green
```

---

### Q30: How do you debug Helm templates?

**Answer:**

**Method 1: helm template (dry-run client-side)**
```bash
# Render templates without installing
helm template myapp ./chart

# With specific values
helm template myapp ./chart -f values-dev.yaml

# Show only specific template
helm template myapp ./chart -s templates/deployment.yaml

# Debug mode (show more info)
helm template myapp ./chart --debug
```

**Method 2: helm install --dry-run**
```bash
# Dry run with server validation
helm install myapp ./chart --dry-run --debug

# See what would be installed
helm install myapp ./chart --dry-run --debug > output.yaml
```

**Method 3: helm lint**
```bash
# Check for issues
helm lint ./chart

# Strict mode
helm lint ./chart --strict
```

**Method 4: Debug within template**
```yaml
# Print debug info to ConfigMap
{{- if .Values.debug }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: debug-{{ .Release.Name }}
data:
  values: |
{{ .Values | toYaml | indent 4 }}
  release: |
{{ .Release | toYaml | indent 4 }}
{{- end }}

# Or use fail for immediate feedback
{{ fail (printf "DEBUG: value is %v" .Values.someValue) }}
```

**Method 5: Common debugging snippets**
```yaml
# Check if value exists
{{ if hasKey .Values "someKey" }}exists{{ else }}missing{{ end }}

# Print type
{{ typeOf .Values.someValue }}

# Print value for inspection  
# VALUE: {{ .Values | toJson }}

# Pretty print
# VALUES:
{{ .Values | toYaml }}
```

---

## Practical Examples

### Example 1: Complete Deployment Template

```yaml
# _helpers.tpl
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}

{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{- with .Values.commonLabels }}
{{ toYaml . }}
{{- end }}
{{- end -}}

{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
{{ include "myapp.labels" . | indent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
{{ include "myapp.selectorLabels" . | indent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
{{ include "myapp.selectorLabels" . | indent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
{{ toYaml . | indent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.fullname" . }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.port }}
          protocol: TCP
        {{- with .Values.livenessProbe }}
        livenessProbe:
{{ toYaml . | indent 10 }}
        {{- end }}
        {{- with .Values.readinessProbe }}
        readinessProbe:
{{ toYaml . | indent 10 }}
        {{- end }}
        resources:
{{ toYaml .Values.resources | indent 10 }}
        {{- with .Values.env }}
        env:
{{ toYaml . | indent 10 }}
        {{- end }}
        {{- if .Values.envFrom }}
        envFrom:
{{ toYaml .Values.envFrom | indent 10 }}
        {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
{{ toYaml . | indent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
{{ toYaml . | indent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
{{ toYaml . | indent 8 }}
      {{- end }}

# values.yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

env: []
envFrom: []

livenessProbe:
  httpGet:
    path: /healthz
    port: http
readinessProbe:
  httpGet:
    path: /ready
    port: http

commonLabels: {}
```

---

### Example 2: Dynamic ConfigMap from Multiple Files

```yaml
# Structure:
# chart/
# ├── config/
# │   ├── app.conf
# │   ├── database.yml
# │   └── cache.json
# └── templates/
#     └── configmap.yaml

# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
  labels:
{{ include "myapp.labels" . | indent 4 }}
data:
{{- $files := .Files }}
{{- range $path, $bytes := $files.Glob "config/*" }}
  {{ base $path }}: |
{{ $files.Get $path | indent 4 }}
{{- end }}
{{- with .Values.additionalConfig }}
{{ toYaml . | indent 2 }}
{{- end }}
```

---

### Example 3: Conditional Resource Creation

```yaml
# ingress.yaml
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "myapp.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ $fullName }}
  labels:
{{ include "myapp.labels" . | indent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
{{ toYaml . | indent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
  {{- range .Values.ingress.tls }}
    - hosts:
      {{- range .hosts }}
        - {{ . | quote }}
      {{- end }}
      secretName: {{ .secretName }}
  {{- end }}
  {{- end }}
  rules:
  {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
        {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ $fullName }}
                port:
                  number: {{ $svcPort }}
        {{- end }}
  {{- end }}
{{- end }}

# values.yaml
ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local
```

---

## Troubleshooting

### Common Error 1: "template: ... :X:Y: executing ... at <.Values...>: nil pointer evaluating"

**Cause:** Accessing a value that doesn't exist

**Solution:**
```yaml
# ❌ Bad
host: {{ .Values.database.mysql.host }}

# ✅ Good - with default
host: {{ .Values.database.mysql.host | default "localhost" }}

# ✅ Good - with conditional
{{- if .Values.database.mysql }}
host: {{ .Values.database.mysql.host }}
{{- end }}

# ✅ Good - with hasKey
{{- if hasKey .Values.database "mysql" }}
host: {{ .Values.database.mysql.host }}
{{- end }}
```

---

### Common Error 2: "template: ... :X:Y: executing ... at <...>: wrong type for value; expected string; got bool"

**Cause:** Type mismatch (e.g., using boolean where string expected)

**Solution:**
```yaml
# ❌ Bad
enabled: {{ .Values.featureEnabled }}
# If featureEnabled is boolean, fails in YAML

# ✅ Good - convert to string
enabled: {{ .Values.featureEnabled | toString | quote }}

# ✅ Or handle as boolean properly
enabled: {{ if .Values.featureEnabled }}true{{ else }}false{{ end }}
```

---

### Common Error 3: YAML indentation errors

**Cause:** Incorrect indentation from toYaml or include

**Solution:**
```yaml
# ❌ Bad
labels:
{{ .Values.labels | toYaml }}

# ✅ Good
labels:
{{ .Values.labels | toYaml | indent 2 }}

# ✅ Better
labels:
{{- .Values.labels | toYaml | nindent 2 }}
```

---

## Best Practices

### 1. ✅ Use Descriptive Template Names

```yaml
# ❌ Bad
{{- define "labels" -}}

# ✅ Good
{{- define "myapp.labels" -}}
```

### 2. ✅ Always Pass Context

```yaml
# ❌ Bad
{{ include "myapp.name" }}

# ✅ Good
{{ include "myapp.name" . }}
```

### 3. ✅ Use `$` for Root Context in Loops

```yaml
{{- range .Values.items }}
name: {{ $.Release.Name }}-{{ . }}
{{- end }}
```

### 4. ✅ Quote String Values

```yaml
# ✅ Good
value: {{ .Values.string | quote }}
```

### 5. ✅ Use `nindent` Instead of `indent`

```yaml
# ✅ Better
labels:
{{- .Values.labels | toYaml | nindent 2 }}
```

### 6. ✅ Validate Required Values

```yaml
apiKey: {{ required "API key is required!" .Values.apiKey }}
```

### 7. ✅ Use Checksum Annotations for ConfigMaps/Secrets

```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

### 8. ✅ Avoid Hardcoding

```yaml
# ❌ Bad
name: my-app-deployment

# ✅ Good
name: {{ include "myapp.fullname" . }}
```

---

## Quick Reference

### Template Syntax
```yaml
{{ }}              # Insert value
{{- }}             # Trim left whitespace
{{ -}}             # Trim right whitespace  
{{- -}}            # Trim both
{{/* comment */}}  # Template comment
```

### Control Structures
```yaml
{{- if CONDITION }}...{{- end }}
{{- if COND }}...{{- else }}...{{- end }}
{{- if COND1 }}...{{- else if COND2 }}...{{- else }}...{{- end }}
{{- range .Items }}...{{- end }}
{{- with .Value }}...{{- end }}
```

### Common Functions
```yaml
quote, squote, upper, lower, title, trim, trunc
default, required, ternary
toYaml, toJson, fromYaml, fromJson
b64enc, b64dec
indent, nindent
include, tpl
```

### Built-in Objects
```yaml
.Values        # values.yaml content
.Release       # Release info
.Chart         # Chart.yaml content  
.Files         # File access
.Capabilities  # Cluster capabilities
```

---

*Happy Helm Templating! 🎯*

*Last Updated: January 2026*
