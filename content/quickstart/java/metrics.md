---
title: "Metrics"
date: 2018-07-16T14:29:27-07:00
draft: false
class: "shadowed-image lightbox"
---

{{% notice note %}}
This guide makes use of Stackdriver for visualizing your data. For assistance setting up Stackdriver, [Click here](/codelabs/stackdriver) for a guided codelab.

It also uses Apache Maven for dependency management and building. If you haven't already installed it, please [Click here](https://maven.apache.org/install.html) for installation instructions
{{% /notice %}}

#### Table of contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Brief Overview](#brief-overview)
- [Getting started](#getting-started)
- [Enable Metrics](#enable-metrics)
    - [Import Packages](#import-metrics-packages)
    - [Create Metrics](#create-metrics)
    - [Create Tags](#create-tags)
    - [Inserting Tags](#inserting-tags)
    - [Recording Metrics](#recording-metrics)
- [Enable Views](#enable-views)
    - [Import Packages](#import-views-packages)
    - [Create Views](#create-views)
    - [Register Views](#register-views)
- [Exporting to Stackdriver](#exporting-to-stackdriver)
    - [Import Packages](#import-exporting-packages)
    - [Export Views](#export-views)
- [Viewing your Metrics on Stackdriver](#viewing-your-metrics-on-stackdriver)

In this quickstart, we’ll gleam insights from code segments and learn how to:

1. Collect metrics using [OpenCensus Metrics](/core-concepts/metrics) and [Tags](/core-concepts/tags)
2. Register and enable an exporter for a [backend](http://localhost:1313/core-concepts/exporters/#supported-backends) of our choice
3. View the metrics on the backend of our choice

#### Requirements
- Java 8+
- [Apache Maven](https://maven.apache.org/install.html)
- Google Cloud Platform account and project
- Google Stackdriver Tracing enabled on your project (Need help? [Click here](/codelabs/stackdriver))

#### Installation
```bash
mvn archetype:generate \
  -DgroupId=io.opencensus.quickstart \
  -DartifactId=repl-app \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false \

cd repl-app/src/main/java/io/opencensus/quickstart

mv App.Java Repl.java
```
Put this in your newly generated `pom.xml` file:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>io.opencensus.quickstart</groupId>
    <artifactId>quickstart</artifactId>
    <packaging>jar</packaging>
    <version>1.0-SNAPSHOT</version>
    <name>quickstart</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <opencensus.version>0.15.0</opencensus.version> <!-- The OpenCensus version to use -->
    </properties>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.5.0.Final</version>
            </extension>
        </extensions>

        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.7.0</version>
                    <configuration>
                        <source>1.8</source>
                        <target>1.8</target>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>appassembler-maven-plugin</artifactId>
                    <version>1.10</version>
                    <configuration>
                        <programs>
                            <program>
                                <id>Repl</id>
                                <mainClass>io.opencensus.quickstart.Repl</mainClass>
                            </program>
                        </programs>
                    </configuration>
                </plugin>
            </plugins>

        </pluginManagement>

    </build>
</project>
```

Put this in `src/main/java/io/opencensus/quickstart/Repl.java`:

```java
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

public class Repl {
    public static void main(String ...args) {
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                System.out.print("> ");
                System.out.flush();
                String line = stdin.readLine();
                String processed = processLine(line);
                System.out.println("< " + processed + "\n");
            } catch (IOException e) {
                System.err.println("Exception "+ e);
            }
        }
    }

    private static String processLine(String line) {
        return line.toUpperCase();
    }
}
```

Install required dependencies:
```bash
mvn install
```

#### Brief Overview
By the end of this tutorial, we will do these four things to obtain metrics using OpenCensus:

1. Create quantifiable metrics (numerical) that we will record
2. Create [tags](/core-concepts/tags) that we will associate with our metrics
3. Organize our metrics, similar to writing a report, in to a `View`
4. Export our views to a backend (Stackdriver in this case)

#### Getting Started
We will be creating a simple "read-evaluate-print" (REPL) app. Let's collect some metrics to observe the work that is going on within this code, such as:

- Latency per processing loop
- Number of lines read
- Number of errors
- Line lengths

Let's first run the application and see what we have.
```bash
mvn exec:java -Dexec.mainClass=io.opencensus.quickstart.Repl
```
You should see something like this:
![java image 1](https://cdn-images-1.medium.com/max/1600/1*VFN-txsDL6qYkN_UH3VwhA.png)

Now, in preparation of collecting metrics, lets abstract some of the core functionality in `main()` to a suite of helper functions:

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

public class Repl {
    public static void main(String ...args) {
        // Step 1. Our OpenCensus initialization will eventually go here

        // Step 2. The normal REPL.
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("Exception "+ e);
            }
        }
    }

    private static String processLine(String line) {
        return line.toUpperCase();
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();
        String line = in.readLine();
        String processed = processLine(line);
        System.out.println("< " + processed + "\n");
    }
}
{{</highlight>}}

#### Enable Metrics

##### Import Packages
To enable metrics, we’ll declare the dependencies in your `pom.xml` file:

{{<tabs Snippet All>}}
{{<highlight xml>}}
<dependencies>
  <dependency>
      <groupId>io.opencensus</groupId>
      <artifactId>opencensus-api</artifactId>
      <version>${opencensus.version}</version>
  </dependency>

  <dependency>
      <groupId>io.opencensus</groupId>
      <artifactId>opencensus-impl</artifactId>
      <version>${opencensus.version}</version>
  </dependency>
</dependencies>
{{</highlight>}}

{{<highlight xml>}}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>io.opencensus.quickstart</groupId>
    <artifactId>quickstart</artifactId>
    <packaging>jar</packaging>
    <version>1.0-SNAPSHOT</version>
    <name>quickstart</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <opencensus.version>0.15.0</opencensus.version> <!-- The OpenCensus version to use -->
    </properties>

    <dependencies>
        <dependency>
            <groupId>io.opencensus</groupId>
            <artifactId>opencensus-api</artifactId>
            <version>${opencensus.version}</version>
        </dependency>

        <dependency>
            <groupId>io.opencensus</groupId>
            <artifactId>opencensus-impl</artifactId>
            <version>${opencensus.version}</version>
        </dependency>
    </dependencies>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.5.0.Final</version>
            </extension>
        </extensions>

        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.7.0</version>
                    <configuration>
                        <source>1.8</source>
                        <target>1.8</target>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>appassembler-maven-plugin</artifactId>
                    <version>1.10</version>
                    <configuration>
                        <programs>
                            <program>
                                <id>Repl</id>
                                <mainClass>io.opencensus.quickstart.Repl</mainClass>
                            </program>
                        </programs>
                    </configuration>
                </plugin>
            </plugins>

        </pluginManagement>

    </build>
</project>
{{</highlight>}}
{{</tabs>}}

Now add the import statements to your `Repl.java`:
{{<tabs Snippet All>}}
{{<highlight java>}}
import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.stats.View;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.stats.View;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;

public class Repl {
    public static void main(String ...args) {
        // Step 1. Our OpenCensus initialization will eventually go here

        // Step 2. The normal REPL.
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("Exception "+ e);
            }
        }
    }

    private static String processLine(String line) {
        return line.toUpperCase();
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();
        String line = in.readLine();
        String processed = processLine(line);
        System.out.println("< " + processed + "\n");
    }
}
{{</highlight>}}
{{</tabs>}}

##### Create Metrics
First, we will create the variables needed to later record our metrics.

{{<tabs Snippet All>}}
{{<highlight java>}}
// The latency in milliseconds
private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

// Counts the number of lines read in from standard input.
private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

// Counts the number of non EOF(end-of-file) errors.
private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

// Counts/groups the lengths of lines read in.
private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

private static final Tagger tagger = Tags.getTagger();
private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

private static void recordStat(MeasureLong ml, Long n) {
    statsRecorder.newMeasureMap().put(ml, n);
}
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.stats.View;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Our OpenCensus initialization will eventually go here

        // Step 2. The normal REPL.
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("Exception "+ e);
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        statsRecorder.newMeasureMap().put(ml, n);
    }

    private static String processLine(String line) {
        return line.toUpperCase();
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();
        String line = in.readLine();
        String processed = processLine(line);
        System.out.println("< " + processed + "\n");
    }
}
{{</highlight>}}
{{</tabs>}}

##### Create Tags
Now we will create the variable later needed to add extra text meta-data to our metrics.

{{<tabs Snippet All>}}
{{<highlight java>}}
// The tag "method"
private static final TagKey KEY_METHOD = TagKey.create("method");
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.stats.View;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    // The tag "method"
    private static final TagKey KEY_METHOD = TagKey.create("method");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Our OpenCensus initialization will eventually go here

        // Step 2. The normal REPL.
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("Exception "+ e);
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        statsRecorder.newMeasureMap().put(ml, n);
    }

    private static String processLine(String line) {
        return line.toUpperCase();
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();
        String line = in.readLine();
        String processed = processLine(line);
        System.out.println("< " + processed + "\n");
    }
}
{{</highlight>}}
{{</tabs>}}

We will later use this tag, called KEY_METHOD, to record what method is being invoked. In our scenario, we will only use it to record that "repl" is calling our data.

Again, this is arbitrary and purely up the user. For example, if we wanted to track what operating system a user is using, we could do so like this:
```java
private static final TagKey OSKey = TagKey.create("operating_system");
```

Later, when we use OSKey, we will be given an opportunity to enter values such as "windows" or "mac".

We will now create helper functions to assist us with recording Tagged Stats.

{{<tabs Snippet All>}}
{{<highlight java>}}
private static void recordTaggedStat(TagKey key, String value, MeasureLong ml, Long n) {
    TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
    try (Scope ss = tagger.withTagContext(tctx)) {
        statsRecorder.newMeasureMap().put(ml, n).record();
    }
}

private static void recordTaggedStat(TagKey key, String value, MeasureDouble md, Double d) {
    TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
    try (Scope ss = tagger.withTagContext(tctx)) {
        statsRecorder.newMeasureMap().put(md, d).record();
    }
}
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.stats.View;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    // The tag "method"
    private static final TagKey KEY_METHOD = TagKey.create("method");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Our OpenCensus initialization will eventually go here

        // Step 2. The normal REPL.
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("Exception "+ e);
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        statsRecorder.newMeasureMap().put(ml, n);
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureLong ml, Long n) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(ml, n).record();
        }
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureDouble md, Double d) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(md, d).record();
        }
    }

    private static String processLine(String line) {
        return line.toUpperCase();
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();
        String line = in.readLine();
        String processed = processLine(line);
        System.out.println("< " + processed + "\n");
    }
}
{{</highlight>}}
{{</tabs>}}

##### Recording Metrics
Finally, we'll hook our stat recorders in to `main`, `processLine`, and `readEvaluateProcessLine`:

{{<tabs Snippet All>}}
{{<highlight java>}}
public static void main(String ...args) {
    // Step 1. Our OpenCensus initialization will eventually go here

    // Step 2. The normal REPL.
    BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

    while (true) {
        try {
            readEvaluateProcessLine(stdin);
        } catch (IOException e) {
            System.err.println("EOF bye "+ e);
            return;
        } catch (Exception e) {
            recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
        }
    }
}

private static String processLine(String line) {
    long startTimeNs = System.nanoTime();

    try {
        return line.toUpperCase();
    } finally {
        long totalTimeNs = System.nanoTime() - startTimeNs;
        double timespentMs = (new Double(totalTimeNs))/1e6;
        recordTaggedStat(KEY_METHOD, "processLine", M_LATENCY_MS, timespentMs);
    }
}

private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
    System.out.print("> ");
    System.out.flush();

    try {
        String line = in.readLine();
        String processed = processLine(line);
        System.out.println("< " + processed + "\n");
        recordStat(M_LINES_IN, new Long(1));
        recordStat(M_LINE_LENGTHS, new Long(line.length()));
    } catch(Exception e) {
        recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
    }
}
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.stats.View;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    // The tag "method"
    private static final TagKey KEY_METHOD = TagKey.create("method");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Our OpenCensus initialization will eventually go here

        // Step 2. The normal REPL.
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("EOF bye "+ e);
                return;
            } catch (Exception e) {
                recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        statsRecorder.newMeasureMap().put(ml, n);
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureLong ml, Long n) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(ml, n).record();
        }
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureDouble md, Double d) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(md, d).record();
        }
    }

    private static String processLine(String line) {
        long startTimeNs = System.nanoTime();

        try {
            return line.toUpperCase();
        } finally {
            long totalTimeNs = System.nanoTime() - startTimeNs;
            double timespentMs = (new Double(totalTimeNs))/1e6;
            recordTaggedStat(KEY_METHOD, "processLine", M_LATENCY_MS, timespentMs);
        }
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();

        try {
            String line = in.readLine();
            String processed = processLine(line);
            System.out.println("< " + processed + "\n");
            recordStat(M_LINES_IN, new Long(1));
            recordStat(M_LINE_LENGTHS, new Long(line.length()));
        } catch(Exception e) {
            recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
        }
    }
}
{{</highlight>}}
{{</tabs>}}

#### Enable Views
In order to examine these stats, we’ll need to export them to the backend of our choice for processing and aggregation.

To do this, we need to define a mechanism for which the backend will process and aggregate those metrics and for this we define Views to categorically describe how we’ll examine the measures.

<a name="import-views-packages"></a>
##### Import Packages

{{<tabs Snippet All>}}
{{<highlight java>}}
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

import io.opencensus.stats.Aggregation;
import io.opencensus.stats.Aggregation.Distribution;
import io.opencensus.stats.BucketBoundaries;
import io.opencensus.stats.Stats;
import io.opencensus.stats.View;
import io.opencensus.stats.View.Name;
import io.opencensus.stats.ViewManager;
import io.opencensus.stats.View.AggregationWindow.Cumulative;
import io.opencensus.tags.TagKey;
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;
import io.opencensus.stats.Aggregation;
import io.opencensus.stats.Aggregation.Distribution;
import io.opencensus.stats.BucketBoundaries;
import io.opencensus.stats.View;
import io.opencensus.stats.View.Name;
import io.opencensus.stats.ViewManager;
import io.opencensus.stats.View.AggregationWindow.Cumulative;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    // The tag "method"
    private static final TagKey KEY_METHOD = TagKey.create("method");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Our OpenCensus initialization will eventually go here

        // Step 2. The normal REPL.
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("Exception "+ e);
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        statsRecorder.newMeasureMap().put(ml, n);
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureLong ml, Long n) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(ml, n).record();
        }
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureDouble md, Double d) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(md, d).record();
        }
    }

    private static String processLine(String line) {
        long startTimeNs = System.nanoTime();

        try {
            return line.toUpperCase();
        } finally {
            long totalTimeNs = System.nanoTime() - startTimeNs;
            double timespentMs = (new Double(totalTimeNs))/1e6;
            recordTaggedStat(KEY_METHOD, "processLine", M_LATENCY_MS, timespentMs);
        }
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();

        try {
            String line = in.readLine();
            String processed = processLine(line);
            System.out.println("< " + processed + "\n");
            recordStat(M_LINES_IN, new Long(1));
            recordStat(M_LINE_LENGTHS, new Long(line.length()));
        } catch(Exception e) {
            recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
        }
    }
}
{{</highlight>}}
{{</tabs>}}

##### Create Views
We now determine how our metrics will be organized by creating `Views`.

{{<tabs Snippet All>}}
{{<highlight java>}}
private static void registerAllViews() {
    // Defining the distribution aggregations
    Aggregation latencyDistribution = Distribution.create(BucketBoundaries.create(
            Arrays.asList(
                // [>=0ms, >=25ms, >=50ms, >=75ms, >=100ms, >=200ms, >=400ms, >=600ms, >=800ms, >=1s, >=2s, >=4s, >=6s]
                0.0, 25.0, 50.0, 75.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0, 2000.0, 4000.0, 6000.0)
            ));

    Aggregation lengthsDistribution = Distribution.create(BucketBoundaries.create(
            Arrays.asList(
                // [>=0B, >=5B, >=10B, >=20B, >=40B, >=60B, >=80B, >=100B, >=200B, >=400B, >=600B, >=800B, >=1000B]
                0.0, 5.0, 10.0, 20.0, 40.0, 60.0, 80.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0)
            ));

    // Define the count aggregation
    Aggregation countAggregation = Aggregation.Count.create();

    // So tagKeys
    List<TagKey> noKeys = new ArrayList<TagKey>();

    // Define the views
    View[] views = new View[]{
        View.create(Name.create("demo/latency"), "The distribution of latencies", M_LATENCY_MS, latencyDistribution, Collections.singletonList(KEY_METHOD)),
        View.create(Name.create("demo/lines_in"), "The number of lines read in from standard input", M_LINES_IN, countAggregation, noKeys),
        View.create(Name.create("demo/errors"), "The number of errors encountered", M_ERRORS, countAggregation, noKeys),
        View.create(Name.create("demo/line_length"), "The distribution of line lengths", M_LINE_LENGTHS, lengthsDistribution, noKeys)
    };

    // Create the view manager
    ViewManager vmgr = Stats.getViewManager();

    // Then finally register the views
    for (View view : views) {
        vmgr.registerView(view);
    }
}
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;
import io.opencensus.stats.Aggregation;
import io.opencensus.stats.Aggregation.Distribution;
import io.opencensus.stats.BucketBoundaries;
import io.opencensus.stats.View;
import io.opencensus.stats.View.Name;
import io.opencensus.stats.ViewManager;
import io.opencensus.stats.View.AggregationWindow.Cumulative;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    // The tag "method"
    private static final TagKey KEY_METHOD = TagKey.create("method");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Our OpenCensus initialization will eventually go here

        // Step 2. The normal REPL.
        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("EOF bye "+ e);
                return;
            } catch (Exception e) {
                recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        statsRecorder.newMeasureMap().put(ml, n);
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureLong ml, Long n) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(ml, n).record();
        }
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureDouble md, Double d) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(md, d).record();
        }
    }

    private static String processLine(String line) {
        long startTimeNs = System.nanoTime();

        try {
            return line.toUpperCase();
        } finally {
            long totalTimeNs = System.nanoTime() - startTimeNs;
            double timespentMs = (new Double(totalTimeNs))/1e6;
            recordTaggedStat(KEY_METHOD, "processLine", M_LATENCY_MS, timespentMs);
        }
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();

        try {
            String line = in.readLine();
            String processed = processLine(line);
            System.out.println("< " + processed + "\n");
            recordStat(M_LINES_IN, new Long(1));
            recordStat(M_LINE_LENGTHS, new Long(line.length()));
        } catch(Exception e) {
            recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
        }
    }

    private static void registerAllViews() {
        // Defining the distribution aggregations
        Aggregation latencyDistribution = Distribution.create(BucketBoundaries.create(
                Arrays.asList(
                    // [>=0ms, >=25ms, >=50ms, >=75ms, >=100ms, >=200ms, >=400ms, >=600ms, >=800ms, >=1s, >=2s, >=4s, >=6s]
                    0.0, 25.0, 50.0, 75.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0, 2000.0, 4000.0, 6000.0)
                ));

        Aggregation lengthsDistribution = Distribution.create(BucketBoundaries.create(
                Arrays.asList(
                    // [>=0B, >=5B, >=10B, >=20B, >=40B, >=60B, >=80B, >=100B, >=200B, >=400B, >=600B, >=800B, >=1000B]
                    0.0, 5.0, 10.0, 20.0, 40.0, 60.0, 80.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0)
                ));

        // Define the count aggregation
        Aggregation countAggregation = Aggregation.Count.create();

        // So tagKeys
        List<TagKey> noKeys = new ArrayList<TagKey>();

        // Define the views
        View[] views = new View[]{
            View.create(Name.create("demo/latency"), "The distribution of latencies", M_LATENCY_MS, latencyDistribution, Collections.singletonList(KEY_METHOD)),
            View.create(Name.create("demo/lines_in"), "The number of lines read in from standard input", M_LINES_IN, countAggregation, noKeys),
            View.create(Name.create("demo/errors"), "The number of errors encountered", M_ERRORS, countAggregation, noKeys),
            View.create(Name.create("demo/line_length"), "The distribution of line lengths", M_LINE_LENGTHS, lengthsDistribution, noKeys)
        };

        // Create the view manager
        ViewManager vmgr = Stats.getViewManager();

        // Then finally register the views
        for (View view : views)
            vmgr.registerView(view);
    }
}
{{</highlight>}}
{{</tabs>}}

##### Register Views
We will create a function called `setupOpenCensusAndStackdriverExporter` and call it from our main function:

{{<tabs Snippet All>}}
{{<highlight java>}}
public static void main(String ...args) {
    // Step 1. Enable OpenCensus Metrics.
    try {
        setupOpenCensusAndStackdriverExporter();
    } catch (IOException e) {
        System.err.println("Failed to create and register OpenCensus Stackdriver Trace exporter "+ e);
        return;
    }

    BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

    while (true) {
        try {
            readEvaluateProcessLine(stdin);
        } catch (IOException e) {
            System.err.println("EOF bye "+ e);
            return;
        } catch (Exception e) {
            recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
        }
    }
}

private static void setupOpenCensusAndStackdriverExporter() throws IOException {
    // Firstly register the views
    registerAllViews();
}
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;
import io.opencensus.stats.Aggregation;
import io.opencensus.stats.Aggregation.Distribution;
import io.opencensus.stats.BucketBoundaries;
import io.opencensus.stats.View;
import io.opencensus.stats.View.Name;
import io.opencensus.stats.ViewManager;
import io.opencensus.stats.View.AggregationWindow.Cumulative;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN= MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    // The tag "method"
    private static final TagKey KEY_METHOD = TagKey.create("method");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Enable OpenCensus Metrics.
        try {
            setupOpenCensusAndStackdriverExporter();
        } catch (IOException e) {
            System.err.println("Failed to create and register OpenCensus Stackdriver Trace exporter "+ e);
            return;
        }

        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("EOF bye "+ e);
                return;
            } catch (Exception e) {
                recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        statsRecorder.newMeasureMap().put(ml, n);
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureLong ml, Long n) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(ml, n).record();
        }
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureDouble md, Double d) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(md, d).record();
        }
    }

    private static String processLine(String line) {
        long startTimeNs = System.nanoTime();

        try {
            return line.toUpperCase();
        } finally {
            long totalTimeNs = System.nanoTime() - startTimeNs;
            double timespentMs = (new Double(totalTimeNs))/1e6;
            recordTaggedStat(KEY_METHOD, "processLine", M_LATENCY_MS, timespentMs);
        }
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();

        try {
            String line = in.readLine();
            String processed = processLine(line);
            System.out.println("< " + processed + "\n");
            recordStat(M_LINES_IN, new Long(1));
            recordStat(M_LINE_LENGTHS, new Long(line.length()));
        } catch(Exception e) {
            recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
        }
    }

    private static void registerAllViews() {
        // Defining the distribution aggregations
        Aggregation latencyDistribution = Distribution.create(BucketBoundaries.create(
                Arrays.asList(
                    // [>=0ms, >=25ms, >=50ms, >=75ms, >=100ms, >=200ms, >=400ms, >=600ms, >=800ms, >=1s, >=2s, >=4s, >=6s]
                    0.0, 25.0, 50.0, 75.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0, 2000.0, 4000.0, 6000.0)
                ));

        Aggregation lengthsDistribution = Distribution.create(BucketBoundaries.create(
                Arrays.asList(
                    // [>=0B, >=5B, >=10B, >=20B, >=40B, >=60B, >=80B, >=100B, >=200B, >=400B, >=600B, >=800B, >=1000B]
                    0.0, 5.0, 10.0, 20.0, 40.0, 60.0, 80.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0)
                ));

        // Define the count aggregation
        Aggregation countAggregation = Aggregation.Count.create();

        // So tagKeys
        List<TagKey> noKeys = new ArrayList<TagKey>();

        // Define the views
        View[] views = new View[]{
            View.create(Name.create("demo/latency"), "The distribution of latencies", M_LATENCY_MS, latencyDistribution, Collections.singletonList(KEY_METHOD)),
            View.create(Name.create("demo/lines_in"), "The number of lines read in from standard input", M_LINES_IN, countAggregation, noKeys),
            View.create(Name.create("demo/errors"), "The number of errors encountered", M_ERRORS, countAggregation, noKeys),
            View.create(Name.create("demo/line_length"), "The distribution of line lengths", M_LINE_LENGTHS, lengthsDistribution, noKeys)
        };

        // Create the view manager
        ViewManager vmgr = Stats.getViewManager();

        // Then finally register the views
        for (View view : views)
            vmgr.registerView(view);
    }

    private static void setupOpenCensusAndStackdriverExporter() throws IOException {
        // Firstly register the views
        registerAllViews();
    }
}
{{</highlight>}}
{{</tabs>}}



#### Exporting to Stackdriver

##### Import Packages

`pom.xml`
{{<tabs Snippet All>}}
{{<highlight xml>}}
<dependency>
    <groupId>io.opencensus</groupId>
    <artifactId>opencensus-exporter-stats-stackdriver</artifactId>
    <version>${opencensus.version}</version>
</dependency>
{{</highlight>}}

{{<highlight xml>}}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>io.opencensus.quickstart</groupId>
    <artifactId>quickstart</artifactId>
    <packaging>jar</packaging>
    <version>1.0-SNAPSHOT</version>
    <name>quickstart</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <opencensus.version>0.15.0</opencensus.version> <!-- The OpenCensus version to use -->
    </properties>

    <dependencies>
        <dependency>
            <groupId>io.opencensus</groupId>
            <artifactId>opencensus-api</artifactId>
            <version>${opencensus.version}</version>
        </dependency>

        <dependency>
            <groupId>io.opencensus</groupId>
            <artifactId>opencensus-impl</artifactId>
            <version>${opencensus.version}</version>
        </dependency>

        <dependency>
            <groupId>io.opencensus</groupId>
            <artifactId>opencensus-exporter-stats-stackdriver</artifactId>
            <version>${opencensus.version}</version>
        </dependency>
    </dependencies>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.5.0.Final</version>
            </extension>
        </extensions>

        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.7.0</version>
                    <configuration>
                        <source>1.8</source>
                        <target>1.8</target>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>appassembler-maven-plugin</artifactId>
                    <version>1.10</version>
                    <configuration>
                        <programs>
                            <program>
                                <id>Repl</id>
                                <mainClass>io.opencensus.quickstart.Repl</mainClass>
                            </program>
                        </programs>
                    </configuration>
                </plugin>
            </plugins>

        </pluginManagement>

    </build>
</project>
{{</highlight>}}
{{</tabs>}}

`Repl.java`
{{<tabs Snippet All>}}
{{<highlight java>}}
import java.io.IOException;

import io.opencensus.exporter.stats.stackdriver.StackdriverStatsConfiguration;
import io.opencensus.exporter.stats.stackdriver.StackdriverStatsExporter;
{{</highlight>}}

{{<highlight java>}}
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import java.io.IOException;

import io.opencensus.exporter.stats.stackdriver.StackdriverStatsConfiguration;
import io.opencensus.exporter.stats.stackdriver.StackdriverStatsExporter;
import io.opencensus.common.Scope;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;
import io.opencensus.stats.Aggregation;
import io.opencensus.stats.Aggregation.Distribution;
import io.opencensus.stats.BucketBoundaries;
import io.opencensus.stats.View;
import io.opencensus.stats.View.Name;
import io.opencensus.stats.ViewManager;
import io.opencensus.stats.View.AggregationWindow.Cumulative;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    // The tag "method"
    private static final TagKey KEY_METHOD = TagKey.create("method");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Enable OpenCensus Metrics.
        try {
            setupOpenCensusAndStackdriverExporter();
        } catch (IOException e) {
            System.err.println("Failed to create and register OpenCensus Stackdriver Trace exporter "+ e);
            return;
        }

        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("EOF bye "+ e);
                return;
            } catch (Exception e) {
                recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        statsRecorder.newMeasureMap().put(ml, n);
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureLong ml, Long n) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(ml, n).record();
        }
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureDouble md, Double d) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(md, d).record();
        }
    }

    private static String processLine(String line) {
        long startTimeNs = System.nanoTime();

        try {
            return line.toUpperCase();
        } finally {
            long totalTimeNs = System.nanoTime() - startTimeNs;
            double timespentMs = (new Double(totalTimeNs))/1e6;
            recordTaggedStat(KEY_METHOD, "processLine", M_LATENCY_MS, timespentMs);
        }
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();

        try {
            String line = in.readLine();
            String processed = processLine(line);
            System.out.println("< " + processed + "\n");
            recordStat(M_LINES_IN, new Long(1));
            recordStat(M_LINE_LENGTHS, new Long(line.length()));
        } catch(Exception e) {
            recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
        }
    }

    private static void registerAllViews() {
        // Defining the distribution aggregations
        Aggregation latencyDistribution = Distribution.create(BucketBoundaries.create(
                Arrays.asList(
                    // [>=0ms, >=25ms, >=50ms, >=75ms, >=100ms, >=200ms, >=400ms, >=600ms, >=800ms, >=1s, >=2s, >=4s, >=6s]
                    0.0, 25.0, 50.0, 75.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0, 2000.0, 4000.0, 6000.0)
                ));

        Aggregation lengthsDistribution = Distribution.create(BucketBoundaries.create(
                Arrays.asList(
                    // [>=0B, >=5B, >=10B, >=20B, >=40B, >=60B, >=80B, >=100B, >=200B, >=400B, >=600B, >=800B, >=1000B]
                    0.0, 5.0, 10.0, 20.0, 40.0, 60.0, 80.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0)
                ));

        // Define the count aggregation
        Aggregation countAggregation = Aggregation.Count.create();

        // So tagKeys
        List<TagKey> noKeys = new ArrayList<TagKey>();

        // Define the views
        View[] views = new View[]{
            View.create(Name.create("demo/latency"), "The distribution of latencies", M_LATENCY_MS, latencyDistribution, Collections.singletonList(KEY_METHOD)),
            View.create(Name.create("demo/lines_in"), "The number of lines read in from standard input", M_LINES_IN, countAggregation, noKeys),
            View.create(Name.create("demo/errors"), "The number of errors encountered", M_ERRORS, countAggregation, noKeys),
            View.create(Name.create("demo/line_length"), "The distribution of line lengths", M_LINE_LENGTHS, lengthsDistribution, noKeys)
        };

        // Create the view manager
        ViewManager vmgr = Stats.getViewManager();

        // Then finally register the views
        for (View view : views)
            vmgr.registerView(view);
    }

    private static void setupOpenCensusAndStackdriverExporter() throws IOException {
        // Firstly register the views
        registerAllViews();
    }
}
{{</highlight>}}
{{</tabs>}}

##### Export Views
We will further expand upon `setupOpenCensusAndStackdriverExporter`:

```java
private static void setupOpenCensusAndStackdriverExporter() throws IOException {
    // Firstly register the views
    registerAllViews();

    String gcpProjectId = envOrAlternative("GCP_PROJECT_ID");

    StackdriverStatsExporter.createAndRegister(
            StackdriverStatsConfiguration.builder()
            .setProjectId(gcpProjectId)
            .build());
}
```

Let's create the helper function `envOrAlternative` to assist with getting the Google Cloud Project ID from environment variable `GCP_PROJECT_ID`:

Here is the final state of the code:
```java
package io.opencensus.quickstart;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

import io.opencensus.common.Scope;
import io.opencensus.stats.Aggregation;
import io.opencensus.stats.Aggregation.Distribution;
import io.opencensus.stats.BucketBoundaries;
import io.opencensus.stats.Stats;
import io.opencensus.stats.Measure;
import io.opencensus.stats.Measure.MeasureLong;
import io.opencensus.stats.Measure.MeasureDouble;
import io.opencensus.stats.Stats;
import io.opencensus.stats.StatsRecorder;
import io.opencensus.stats.View;
import io.opencensus.tags.Tags;
import io.opencensus.tags.Tagger;
import io.opencensus.tags.TagContext;
import io.opencensus.tags.TagContextBuilder;
import io.opencensus.tags.TagKey;
import io.opencensus.tags.TagValue;
import io.opencensus.stats.View;
import io.opencensus.stats.View.Name;
import io.opencensus.stats.ViewManager;
import io.opencensus.stats.View.AggregationWindow.Cumulative;
import io.opencensus.exporter.stats.stackdriver.StackdriverStatsConfiguration;
import io.opencensus.exporter.stats.stackdriver.StackdriverStatsExporter;

public class Repl {
    // The latency in milliseconds
    private static final MeasureDouble M_LATENCY_MS = MeasureDouble.create("repl/latency", "The latency in milliseconds per REPL loop", "ms");

    // Counts the number of lines read in from standard input.
    private static final MeasureLong M_LINES_IN = MeasureLong.create("repl/lines_in", "The number of lines read in", "1");

    // Counts the number of non EOF(end-of-file) errors.
    private static final MeasureLong M_ERRORS = MeasureLong.create("repl/errors", "The number of errors encountered", "1");

    // Counts/groups the lengths of lines read in.
    private static final MeasureLong M_LINE_LENGTHS = MeasureLong.create("repl/line_lengths", "The distribution of line lengths", "By");

    // The tag "method"
    private static final TagKey KEY_METHOD = TagKey.create("method");

    private static final Tagger tagger = Tags.getTagger();
    private static final StatsRecorder statsRecorder = Stats.getStatsRecorder();

    public static void main(String ...args) {
        // Step 1. Enable OpenCensus Metrics.
        try {
            setupOpenCensusAndStackdriverExporter();
        } catch (IOException e) {
            System.err.println("Failed to create and register OpenCensus Stackdriver Trace exporter "+ e);
            return;
        }

        BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            try {
                readEvaluateProcessLine(stdin);
            } catch (IOException e) {
                System.err.println("EOF bye "+ e);
                return;
            } catch (Exception e) {
                recordTaggedStat(KEY_METHOD, "repl", M_ERRORS, new Long(1));
                return;
            }
        }
    }

    private static void recordStat(MeasureLong ml, Long n) {
        TagContext tctx = tagger.emptyBuilder().build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(ml, n).record();
        }
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureLong ml, Long n) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(ml, n).record();
        }
    }

    private static void recordTaggedStat(TagKey key, String value, MeasureDouble md, Double d) {
        TagContext tctx = tagger.emptyBuilder().put(key, TagValue.create(value)).build();
        try (Scope ss = tagger.withTagContext(tctx)) {
            statsRecorder.newMeasureMap().put(md, d).record();
        }
    }

    private static String processLine(String line) {
        long startTimeNs = System.nanoTime();

        try {
            return line.toUpperCase();
        } catch (Exception e) {
            recordTaggedStat(KEY_METHOD, "processLine", M_ERRORS, new Long(1));
            return "";
        } finally {
            long totalTimeNs = System.nanoTime() - startTimeNs;
            double timespentMs = (new Double(totalTimeNs))/1e6;
            recordTaggedStat(KEY_METHOD, "processLine", M_LATENCY_MS, timespentMs);
        }
    }

    private static void readEvaluateProcessLine(BufferedReader in) throws IOException {
        System.out.print("> ");
        System.out.flush();

        String line = in.readLine();
        String processed = processLine(line);
        System.out.println("< " + processed + "\n");
        if (line != null && line.length() > 0) {
            recordStat(M_LINES_IN, new Long(1));
            recordStat(M_LINE_LENGTHS, new Long(line.length()));
        }
    }

    private static void registerAllViews() {
        // Defining the distribution aggregations
        Aggregation latencyDistribution = Distribution.create(BucketBoundaries.create(
                Arrays.asList(
                    // [>=0ms, >=25ms, >=50ms, >=75ms, >=100ms, >=200ms, >=400ms, >=600ms, >=800ms, >=1s, >=2s, >=4s, >=6s]
                    0.0, 25.0, 50.0, 75.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0, 2000.0, 4000.0, 6000.0)
                ));

        Aggregation lengthsDistribution = Distribution.create(BucketBoundaries.create(
                Arrays.asList(
                    // [>=0B, >=5B, >=10B, >=20B, >=40B, >=60B, >=80B, >=100B, >=200B, >=400B, >=600B, >=800B, >=1000B]
                    0.0, 5.0, 10.0, 20.0, 40.0, 60.0, 80.0, 100.0, 200.0, 400.0, 600.0, 800.0, 1000.0)
                ));

        // Define the count aggregation
        Aggregation countAggregation = Aggregation.Count.create();

        // So tagKeys
        List<TagKey> noKeys = new ArrayList<TagKey>();

        // Define the views
        View[] views = new View[]{
            View.create(Name.create("demo/latency"), "The distribution of latencies", M_LATENCY_MS, latencyDistribution, Collections.singletonList(KEY_METHOD)),
            View.create(Name.create("demo/lines_in"), "The number of lines read in from standard input", M_LINES_IN, countAggregation, noKeys),
            View.create(Name.create("demo/errors"), "The number of errors encountered", M_ERRORS, countAggregation, Collections.singletonList(KEY_METHOD)),
            View.create(Name.create("demo/line_lengths"), "The distribution of line lengths", M_LINE_LENGTHS, lengthsDistribution, noKeys)
        };

        // Create the view manager
        ViewManager vmgr = Stats.getViewManager();

        // Then finally register the views
        for (View view : views)
            vmgr.registerView(view);
    }

    private static void setupOpenCensusAndStackdriverExporter() throws IOException {
        // Firstly register the views
        registerAllViews();

        // Make sure to set the environment variable "GCP_PROJECT_ID"
        String gcpProjectId = envOrAlternative("GCP_PROJECT_ID");

        StackdriverStatsExporter.createAndRegister(
                StackdriverStatsConfiguration.builder()
                .setProjectId(gcpProjectId)
                .build());
    }

    private static String envOrAlternative(String key, String ...alternatives) {
        String value = System.getenv().get(key);
        if (value != null && value != "")
            return value;

        // Otherwise now look for the alternatives.
        for (String alternative : alternatives) {
            if (alternative != null && alternative != "") {
                value = alternative;
                break;
            }
        }

        return value;
    }
}
```

#### Viewing your Metrics on Stackdriver
With the above you should now be able to navigate to the [Google Cloud Platform console](https://app.google.stackdriver.com/metrics-explorer), select your project, and view the metrics.

In the query box to find metrics, type `quickstart` as a prefix:

![viewing metrics 1](/img/quickstart-metrics-available.png)

And on selecting any of the metrics e.g. `OpenCensus/demo/lines_in`, we’ll get...

![viewing metrics 2](/img/quickstart-metrics-java-lines_in.png)


On checking out the Stacked Area display of the latency, we can see that the 99th percentile latency was 24.75ms.

![Latency](/img/quickstart-metrics-java-latency.png)

Latency heatmap
![Latency heatmap](/img/quickstart-metrics-java-latency-heatmap.png)

And, for `line_lengths`:
![Line lengths](/img/quickstart-metrics-java-line-length-rate.png)
