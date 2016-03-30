# Basics

## License

Each source file has to specify the license at the very top of the file:

```
/*
 * Copyright (c) 2016 VMware, Inc. All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not
 * use this file except in compliance with the License.  You may obtain a copy of
 * the License at http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software distributed
 * under the License is distributed on an "AS IS" BASIS, without warranties or
 * conditions of any kind, EITHER EXPRESS OR IMPLIED.  See the License for the
 * specific language governing permissions and limitations under the License.
 */
```

License header file: [contrib/header.txt](https://github.com/vmware/xenon/blob/master/contrib/header.txt)


## Line length

Set to `100`

## Import statements

The order of import statements is:

- import static `java.*`
- import static `javax.*`
- blank line
- import static all other imports
- blank line
- import static `com.vmware.*`
- blank line
- import `java.*`
- import `javax.*`
- blank line
- import all other imports
- blank line
- import `com.vmware.*`


## Comments

Comments are always placed at independent line.  
Do not append the comment at the end of the line.

**Wrong**
```java
  host.setLoggingLevel(Level.OFF);  // setting log level OFF
```

**Correct**
```java
  // setting log level OFF
  host.setLoggingLevel(Level.OFF);
```

## Commit message

Follow the widely used format:

- [Commit Guidelines section of Pro Git](https://git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project#Commit-Guidelines)  
- ["Shiny new commit styles" from Github blog](https://github.com/blog/926-shiny-new-commit-styles)


**Sample:**
```
Short (50 chars or less) summary of changes

More detailed explanatory text, if necessary.  Wrap it to
about 72 characters or so.  In some contexts, the first
line is treated as the subject of an email and the rest of
the text as the body.  The blank line separating the
summary from the body is critical (unless you omit the body
entirely); tools like rebase can get confused if you run
the two together.

Further paragraphs come after blank lines.

  - Bullet points are okay, too

  - Typically a hyphen or asterisk is used for the bullet,
    preceded by a single space, with blank lines in
    between, but conventions vary here
```

- 50 char in title
- Wrap the body at 72 char or less





## Checkstyle

Checkstyle runs as part of maven `validate` lifecycle.

You can call it manually like `./mvnw validate` or `./mvnw checkstyle:checkstyle`.

checkstyle file: [checkstyle.xml](https://github.com/vmware/xenon/blob/master/checkstyle.xml)


# IDE Settings

## Formatter

For both Eclipse and IntellJ, import [contrib/eclipse-java-style.xml](https://github.com/vmware/xenon/blob/master/contrib/eclipse-java-style.xml)

### IntelliJ

IntelliJ can import eclipse formatter file.

`Preference` - `Editor` - `Code Style` - `Manage` - `Import`

Import `contrib/eclipse-java-style.xml`.

Once "VMware DCP" is imported, select it and click "Copy to Project"


## IntelliJ Specific

### Setting java package import order

1. Update `.idea/codeStyleSettings.xml` with [contrib/idea-java-style.xml](https://github.com/vmware/xenon/blob/master/contrib/idea-java-style.xml)
1. Restart IntelliJ
