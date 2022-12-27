# Helm templates: do not use tpl in vain

From the documentation:

> *The* `tpl` function allows developers to evaluate strings as templates inside a template. This is useful to pass a template string as a value to a chart or render external configuration files. Syntax: `{{ tpl TEMPLATE_STRING VALUES }}`

When writing generic Helm charts or libraries, **calls to** `tpl` are often overused, and for a good reason: they give lots of flexibility, and allow chart users to avoid repetition in the `values.yaml` file.

The `tpl` function is however costly and slow, and this is why it should be called only when necessary.

%[https://github.com/helm/helm/issues/8002] 

**How to limit calls while keeping the flexibility?** My solution is to wrap all calls to `tpl` using the following helper function:

```go
{{- define "bettertpl" -}}
  {{- $tpl := .value -}}
  {{- /* handle cases where .value is a yaml object */ -}}
  {{- if not (typeIs "string" $tpl) -}}
    {{- $tpl = toYaml $tpl -}}
  {{- end -}}
  {{- /* only call tpl if there is at least one template expression */ -}}
  {{- if contains "{{" $tpl -}}
    {{- tpl $tpl .context }}
  {{- else -}}
    {{- $tpl -}}
  {{- end -}}
{{- end -}}
```

As the code should make it clear, `bettertpl` adds two features to the regular `tpl`:

1.  it allows to template anything (not just string), as `dict`, `list`, etc. will be converted to string first, and
    
2.  it only calls the slow `tpl` when needed: if the string doesn't contain at least one `{{ ... }}`, we know we can just print it as is.
    

Usage:

```go
before: {{ tpl .Values.foo . }}
after: {{ include "bettertpl" (dict "value" .Values.foo "context" .) }}
```

**Do you find it too verbose?**

Note that I use named arguments for better readability. You can get rid of them and use lists instead:

```go
{{ include "bettertpl" (list .Values.foo .) }}
```

In the template above, change `.value` → `first .` and `.context` → `index . 2` to read from list arguments instead. {%endcollapsible %}

* * *

**Is it worth it?** As an example, I recently migrated a gitops repository with an umbrella chart of around 20 sub-charts. After a refactoring allowing me to use only one base chart for all, my `helm template` went from &lt;1s to more than 15 seconds...

I was able to take it down to 4 seconds by limiting the calls to `tpl` with this simple trick, which is quite an improvement.