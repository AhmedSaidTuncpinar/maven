<!---
 Licensed to the Apache Software Foundation (ASF) under one or more
 contributor license agreements.  See the NOTICE file distributed with
 this work for additional information regarding copyright ownership.
 The ASF licenses this file to You under the Apache License, Version 2.0
 (the "License"); you may not use this file except in compliance with
 the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

### Overview

Cache configuration provides you additional control over incremental maven behavior. Follow it step by step to
understand how it works and figure out your optimal config

### Minimal config

Absolutely minimal config which enables incremental maven with local cache

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<cache xmlns="org:apache:maven:cache:config:v1"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="org:apache:maven:cache:config:v1 cache-config.xsd">

    <configuration>
        <enabled>true</enabled>
        <hashAlgorithm>XX</hashAlgorithm>
    </configuration>

    <input>
        <global>
            <glob>{*.java,*.xml,*.properties}</glob>
        </global>
    </input>
</cache>
```

### Enabling remote cache

Just add `<remote>` section under `<configuration>`

```xml

<configuration>
    <enabled>true</enabled>
    <hashAlgorithm>XX</hashAlgorithm>
    <remote>
        <url>https://yourserver:port</url>
    </remote>
</configuration>
```

### Adding more file types to input

Add all the project specific source code files in `<glob>`. Scala in this case:

```xml

<input>
    <global>
        <glob>{*.java,*.xml,*.properties,*.scala}</glob>
    </global>
</input>
```

### Adding source directory for bespoke project layouts

In most of the cases incremental maven will recognize directories automatically by build introspection. If not, you can
add additional directories with `<include>`. Also you can filter out undesirable dirs and files by using exclude tag

```xml

<input>
    <global>
        <glob>{*.java,*.xml,*.properties,*.scala}</glob>
        <include>importantdir/</include>
        <exclude>tempfile.out</exclude>
    </global>
</input>
```

### Plugin property is env specific (breaks checksum and caching)

Consider to exclude env specific properties:

```xml

<input>
    <global>
        ...
    </global>
    <plugin artifactId="maven-surefire-plugin">
        <effectivePom>
            <excludeProperty>argLine</excludeProperty>
        </effectivePom>
    </plugin>
</input>
```

Implications - builds with different `argLine` will have identical checksum. Validate that is semantically valid.

### Plugin property points to directory where only subset of files is relevant

If plugin configuration property points to `somedir` it will be scanned with default glob. You can tweak it with custom
processing rule

```xml

<input>
    <global>
        ...
    </global>
    <plugin artifactId="protoc-maven-plugin">
        <dirScan mode="auto">
            <!--<protoBaseDirectory>${basedir}/..</protoBaseDirectory>-->
            <tagScanConfig tagName="protoBaseDirectory" recursive="false" glob="{*.proto}"/>
        </dirScan>
    </plugin>
</input>
```

### Local repository is not updated because `install` is cached

Add `executionControl/runAlways` section

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<cache xmlns="org:apache:maven:cache:config:v1" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="org:apache:maven:cache:config:v1 cache-config.xsd">
    <configuration>
        ...
    </configuration>
    <input>
        ...
    </input>
    <executionControl>
        <runAlways>
            <plugin artifactId="maven-failsafe-plugin"/>
            <execution artifactId="maven-dependency-plugin">
                <execId>unpack-autoupdate</execId>
            </execution>
            <goals artifactId="maven-install-plugin">
                <goal>install</goal>
            </goals>
        </runAlways>
    </executionControl>
</cache>
``` 

### I occasionally cached build with `-DskipTests=true` and tests do not run now

If you add command line flags to your build, they do not participate in effective pom - maven defers final value
resolution to plugin runtime. To invalidate build if filed value is different in runtime, add reconciliation section
to `executionControl`:

```xml

<executionControl>
    <runAlways>
        ...
    </runAlways>
    <reconcile>
        <plugin artifactId="maven-surefire-plugin" goal="test">
            <reconcile propertyName="skip" skipValue="true"/>
            <reconcile propertyName="skipExec" skipValue="true"/>
            <reconcile propertyName="skipTests" skipValue="true"/>
            <reconcile propertyName="testFailureIgnore" skipValue="true"/>
        </plugin>
    </reconcile>
</executionControl>
```

Please notice `skipValue` attribute. It denotes value which forces skipped execution.
Read `propertyName="skipTests" skipValue="true"` as if property skipTests has value true, plugin will skip execution If
you declare such value incremental maven will reuse appropriate full-build though technically they are different, but
because full-build is better it is safe to reuse

### How to renormalize line endings in working copy after committing .gitattributes (git 2.16+)

Ensure you've committed (and ideally pushed everything) - no changes in working copy. After that:

```shell
# Rewrite objects and update index
git add --renormalize .
# Commit changes
git commit -m "Normalizing line endings"
# Remove working copy paths from git cache
git rm --cached -r .
# Refresh with new line endings
git reset --hard
```

### I want to cache interim build and override it later with final version

Solution: set `-Dremote.cache.save.final=true` to nodes which produce final builds. Such builds will not be overridden
and eventually will replace all interim builds