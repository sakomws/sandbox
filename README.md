# Helm cmds

## Create new chart:
helm create mychart

# Add file in /templates called mychart/templates/configmap.yaml:
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
  
# Prints all Kube resources uploaded to tiller:
helm get manifest full-coral(release name)

# Delete release:
helm delete full-coral

# Test template rendering but not install:
helm install --debug --dry-run ./mychart

# Release object inside:
.Name: Rel Name
.Time: Time of Rel
.Namespace: namespace to be released into(if manifest not override)
.Service: Name of rel service(always tiller)
.Revision: Revision number of rel, begin at 1 and increments for each helm upgrade
.IsUpgrade: Set to true if operation is upgrade or rollback
.IsInstall: Set to true if operation is install

# Values object passed from values.yaml, default is empty. It comes from 4 sources:
1. values.yaml file in chart
2. if subchart, values.yaml file of parent chart
3. Values file passed into 'helm install' or 'helm upgrade' with -f flag:
helm install -f myvals.yaml ./mychart
4. Individual parameters passed with --set:
helm install --set foo=bar ./mychart

Example:
helm install --dry-run --debug ./mychart
helm install --dry-run --debug --set favoriteDrink=slurm ./mychart

To delete key from values:
helm install --debug --dry-run ../mychart --set favorite.drink=null

# Chart: Content of Chart.yaml

# Files: Access to non-special files in a chart:
.Get: Get file by name(.Files.Get config.ini)
.GetBytes Get file content as array of bytes instead of string. Useful for images

# Capabilities: K8s capabilities supported:
.APIVersions: Set of versions
APIVersions.Has $version: indicates whether version(batch/v1) is enabled on cluster
.KubeVersion: To look at Kube values: Major, Minor,GitVersion,GitCommit,GitTreeState,BuildDate,GoVersion,Compiler and Platform
.TillerVersion: To look at Tiller version: SemVer,GitCommit,GitTreeState

# Template: Contain info about current template executed:
.Name: namespaced filepath to current template
.BasePath: namespaced path to template dir of current chart

# Functions, functionName arg1 arg2... style:
quote: ""
upper:
default "tea"
repeat N
indent N "k:v"
title: Case function, make 1st letter big
|- : declare multiline string in YAML, 
tuple: 
List-like collection of fix size and arbitrary data types. 
To make list inside template and iterate over it.
range: Can be used to iterate over k/v collections like map or dict, or tuple.

# Pipelines
food: {{ .Values.favorite.food | upper | quote }}
food:"PIZZA"
food: {{ .Values.favorite.food | repeat 5 | quote }}
food: "pizzapizzapizza..."
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}


Note1: Builtin values are always begin with capital letter.
Note2: Default commands is perfect for computed values, which cannot be inside values.yaml

# Liveness probe:
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120

To overrite livenessProbe handler from exec to httpGet:
--set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt]

To delete the livenessProbe.httpGet, because K8s supports only 1 Probe handler:
helm install stable/drupal 
--set image=my-registry/drupal:0.1.0 
--set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] 
--set livenessProbe.httpGet=null

# Operators and funcs
eq,ne,lt,gt,and,or

# Flow control
if/else: to create conditional blocks
with: specify scope
range: which provides "for each" style loop
define: declares new named template inside ur template
template: imports named template
block: declares special kind of fillable template area

# IF/ELSE
{{if PIPELINE}}
 #do smt
{{ else if OTHEr PIPELINE}}
 #do smt else
{{else}}
 #default case
{{end}}

Control structures can execute entire pipeline, not just evaluate a value. 
Pipeline is false if the value is:
-boolean false
-numeric zero
-empty string
-a nil(empty or null)
-an empty collection(map,slice,tuple,dict,array)
{{- 3 }}: trim left whitespace and print 3
{{-3}}: print "-3"
*{{}}: Indicates newline character will be removed

# Modify scope using WITH
. is a reference to current scope.
Scopes can be changed using 'with' to set current scope to particular one. For ex:
{{- with .Values.favorite}}
drink:{{.drink | default "tea" | quote}}
food: {{.food | upper | quote}}
{{- end}}
The scope is reset to previous scope after 'end' statement.
Inside restricted scope, wont able to access objects from parent scopes: NOO: .Release.Name

# Looping with RANGE action
toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
OR:
 sizes: |-
    {{- range tuple "small" "medium" "large" }}
    - {{ . }}
    {{- end }}

# Variables
$ - point to root context, is always global

# Named Templates(partial or subtemplate), 2 ways create, few ways to use them:
Declare and manage temps: define,template, block
Special purpose 'include' func that works similar to template action.
# Template names are global

Template names are global. If declare 2 temps w same name, last loaded will be used.
Bec, temps in subcharts are compiled together w top-level temps.
Naming convention: Prefix each defined template w name of chart: {{ define "mychart.labels"}}
To avoid 2 diff charts implement temps of same name.

# Partials and _ FILES

templates/ : Kube manifests
_dsdas: _ as start not have Kube manifests, but avail everywhere within other chart temps use.
helpers.tpl: default loc for template partials
{{define "MY.NAME"}}
 # body of temp here
{{ end }}

-Inside base template:
{{- template "mychart.labels" }}

# Set scope of template
Pass scope to temp:
 {{- template "mychart.labels" . }}

=====
Charts must be smaller than 1mb for Kube obj.
Some files  cannot be accessed through .Files obj for sec reason:
-files in templates/
-files excluded in .helmignore

Subcharts:
-Considered stand-alone, never explicitly depen on parent chart
-Subchart cannot acces values of its parent
-Parent chart can override values for subcharts
-Helm has concept of global values that can accessed by all charts

--
cd mychart/charts
helm create mysubchart
rm -rf mysubchart/templates/*.*

values.yaml in mychart/charts/mysubchart
dessert: cake

New configmap template in:
mychart/charts/mysubchart/templates/configmap.yaml

helm install --dry-run --debug mychart/charts/mysubchart
!!str
!!int
3 'inline' ways to declare str:
w1: bare words
w2: "double quoted str"
w3: 'sing quoted str'

Multiline str:
coffe: |
 sako
 maka
 tako

# Inject content of file into template:
use {{.Files.Get "Filename"}}
use {{ include "Template" . }}

# Insert static file is todo smt. like this:
myfile: |
{{.Files.Get "myfile.txt" | indent 2}}

# Folded multi-line str
Represent YAML w multi lines, treated as 1 long line when interpreted.

If we want YAML processor to strip off the trailing newline, add
a - after the | :

All inline styles must be on one line
