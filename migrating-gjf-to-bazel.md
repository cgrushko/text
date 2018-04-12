# Migrating google-java-format from Maven to Bazel using Jadep

[Jadep](https://github.com/bazelbuild/tools_jvm_autodeps/tree/master/jadep) is a
tool to incrementally update Bazel BUILD files based on what Java files need, and
is generally intended to be used when a project is already built using Bazel, but
it can also be used to bootstrap a project.

In a nutshell, it's run like that:

```bash
# Adds missing `deps` in the BUILD rule has `srcs = [Foo.java]`.
~/bin/jadep src/main/java/com/Foo.java

# Adds missing `deps` in the BUILD rule named //src/main/java/com:Foo,
# based on all Java files its `srcs` attribute contains.
~/bin/jadep //src/main/java/com:Foo
```

Jadep respects existing BUILD files, but will create new ones when needed. We can
therefore guide Jadep with a pre-existing structure.

The result can be found in <https://github.com/cgrushko/google-java-format/tree/bazel>.

In this migration, I took these steps:

0. Create a Bazel repository.
1. Set up third-party precompiled Jars from Maven Central. This is done using <https://github.com/johnynek/bazel-deps>.
2. Start Bazel and fetch Jars.
3. Create a skeleton of BUILD files and Run Jadep
4. Add a `java_binary` rule, and required resources to tests.

_Note:_ I didn't migrate the Eclipse and IntelliJ sub-projects.

## Create a Bazel repository

Bazel uses a file named WORKSPACE to find the root of its world.

```bash
echo > WORKSPACE
```

## Third-party Jars from Maven Central

Bazel's [`maven_jar`](https://docs.bazel.build/versions/master/be/workspace.html#maven_jar) downloads Jars and makes them available to a Bazel build. However, it doesn't handle transitive dependencies.

<https://github.com/johnynek/bazel-deps> is a complementary tool that walks a Jar's dependency graph and creates BUILD files to reflect it.

A YAML file is used to configure `bazel-deps`; here, I grep-ed google-java-format's `pom.xml` files and manually
populated the YAML file, though I'm sure there's an automated method.

The resulting files are shown in <https://github.com/cgrushko/google-java-format/commit/6bc90d0fc9d906727db385826d3fccd640489741>.
A snippet from that commit:

```
dependencies:
  junit:
    junit:
      lang: java
      version: "4.12"
```

To run `bazel-deps`, I used

    $ bazel run //:parse -- generate -r ~/code/google-java-format-bazel -s thirdparty/maven.bzl -d maven_deps.yaml

## Start Bazel and fetch Jars

```bash
bazel build
bazel fetch ...
```

## Create a skeleton of BUILD files and run Jadep

Given the following BUILD file,

```python
java_library(
    name = "googlejavaformat",
    srcs = glob(["*.java"]),
)
```

running Jadep as follows will add all missing `deps` that the Java files in this
directory need to build successfully.

```bash
~/bin/jadep //path/to:googlejavaformat
```

If Jadep is run on a Java file without a matching rule, Jadep will create one.
A heuristic is used to determine the kind of the rule (java_library, java_test, etc.).

I chose the following structure for this project:

1. Every Java code package is in a single `java_library` rule.
2. Every Java test class is in its own `java_test` rule. (`java_test` rules only support a single JUnit test class anyway.)

For code packages, I created BUILD files like the one at the top of this section.
For test classes, I didn't do anything and let Jadep create new rules when needed.

Finally, I ran Jadep on all code packages and on all test .java files.

The script I used is https://github.com/cgrushko/google-java-format/commit/4f4cb535edf0d19e08a71fb2be15c44dcf6becfd, and a snippet is:

```python
# Snippet - do not run
for code_dir in code_dirs:
  with open(code_dir + "/BUILD", 'w') as f:
    f.write("package(default_visibility = ['//:__subpackages__'])\n" +
            "java_library(name = '{}', srcs = glob(['*.java']))".format(os.path.basename(code_dir)))

cmd = [os.path.expanduser("~/bin/jadep"), '-content_roots='+code_root] + ['//'+x for x in code_dirs]
print (cmd)
call(cmd)
```

### Deep dive: content roots

tl;dr: "content roots" are usually `src/main/java` and `src/test/java`, are
the default in Jadep, and can be set using the `-content_roots` flag.

One method Jadep uses to find Java dependencies is to guess the location of the
file defining a class. For example, `com.foo.Bar` should be in a file called
`com/foo/Bar.java`.

"Content roots" are prefixed to a transformed class name to form a path from the
workspace root to the file. In the example above, `com.foo.Bar` can either be in
`src/main/java/com/foo/Bar.java` or in `src/test/java/com/foo/Bar.java`.

Jadep searches for files in all the content roots.

## Mimicking Maven's "replacement", adding a `java_binary` rule, and adding required resources to tests

The file `GoogleJavaFormatVersion.java` is generated from `GoogleJavaFormatVersion.java.template`
by replacing `%VERSION%` with the current version. In order to do this in Bazel, we use a [`genrule`](https://docs.bazel.build/versions/master/be/general.html#genrule):

```python
VERSION = "1.6-SNAPSHOT"

genrule(
    name = "GoogleJavaFormatVersion",
    srcs = ["GoogleJavaFormatVersion.java.template"],
    outs = ["GoogleJavaFormatVersion.java"],
    cmd = "sed 's/%VERSION%/" + VERSION + "/' $< > $@ ",
)
```

We add it to the list of `srcs` that the rule in the same package consumes.

The project's `Main.java` file, which contains the `main()` method, is actually
used in tests. Since `java_test` can't depend on a `java_binary`, the `Main.java`
file must be in a `java_library` rule.

In order to build an executable Jar, we wrap the main `java_library` in a `java_binary`:

```python
java_binary(
    name = "Main",
    runtime_deps = [":java"],
)
```

The resulting commit is <https://github.com/cgrushko/google-java-format/commit/c822e3af11af2b783d1d81fd9d431da087f26840.>

At this point Bazel is able to build google-java-format:

```bash
bazel build //core/src/main/java/com/google/googlejavaformat/java:Main_deploy.jar
java -jar bazel-bin/core/src/main/java/com/google/googlejavaformat/java/Main_deploy.jar
```

Running the tests shows that some are failing due to missing resources. This part
can't easily be automated, unfortunately.  Adding the required resources manually (commit <https://github.com/cgrushko/google-java-format/commit/9e55f1ee2a04a2923ca1f4511227e21591cc2e14>) makes the tests pass.
(except one, which is a `@Parameterized` test, which `java_test` fails to run.)
