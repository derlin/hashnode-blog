# How to create jars that run like any other executable binary (./app.jar)

Fat jars are a good way to package java applications, whether they are command-line programs or GUIs. However, a jar differs from other executables: instead of the regular `./app.jar`, it must be invoked using `java -jar app.jar`. This is ok, but not ideal.

It is not a given though: [Spring Boot](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/deployment-install.html) is able to generate ***executable jars***, that is jars that can be executed using the direct syntax `./app.jar` like any other executable binary. How do they pull this off? And, more importantly, how can we apply the same logic to any jar? Let's find out!

*Why not a native executable 😐 ?*

An even better way is to create a real *native executable* using [GraalVM](https://www.graalvm.org), which directly embeds a tiny Virtual Machine, so it can run even on machines that do not have a JRE installed. However, this process is tedious and has limitations... It won't work for any codebase! If you assume all your users will have a JRE, this solution is way easier.

## The magic behind executable jars

The actual magic involved is pretty straightforward and based on a little-known fact about the Zip format. From [the .ZIP format wiki page](https://en.wikipedia.org/wiki/ZIP_(file_format)#Combination_with_other_file_formats):

> *The .ZIP file format allows for a comment containing up to 65,535 (216−1) bytes of data to occur at the end of the file after the central directory* \[...\]
> 
> *This allows arbitrary data to occur in the file both before and after the ZIP archive data, and for the archive to still be read by a ZIP application. A side-effect of this is that it is possible to author a file that is both a working ZIP archive and another format, provided that the other format tolerates arbitrary data at its end, beginning, or middle.*

Since JAR is a variant of ZIP, it works for them as well. It means it is possible to append a bash script, acting like a launcher, at the beginning of a jar file without corrupting it.

This is exactly what Spring Boot does. Take any executable Spring Boot jar (for example [bbdata-api-\*.jar](https://github.com/big-building-data/bbdata-api/releases/tag/nightly)), and run `head` on it. You should see:

```bash
head -n 10 /tmp/bbdata-api-2.0.0-alpha.jar
#!/bin/bash
#
#    .   ____          _            __ _ _
#   /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
#  ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
#   \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
#    '  |____| .__|_| |_|_| |_\__, | / / / /
#   =========|_|==============|___/=/_/_/_/
#   :: Spring Boot Startup Script ::
#
```

## Turning a jar into an executable (with one bash command !)

With this trick in mind, turning any jar into an executable jar is as easy as running those two commands (see [this gist](https://gist.github.com/joewalnes/e200c21288edaa970453ec47b6711254) for a variant):

```bash
# Append a basic launcher script to the jar
cat \
  <(echo '#!/bin/sh')\
  <(echo 'exec java -jar $0 "$@"')\
  <(echo 'exit 0')\
  original.jar > executable.jar

# Make the new jar executable
chmod +x executable.jar
```

And it **works on all unix-like systems** including Linux, MacOS, Cygwin, and Windows Linux subsystem!

## Making executable jars using Gradle

Now that the process is understood, writing a Gradle Task for it is easy.

### Custom Gradle Task

First, we need to define a new custom task in `build.gradle.kts`:

```kotlin
abstract class ExecutableJarTask: DefaultTask() {
    // This custom task will prepend the content of a
    // bash launch script at the beginning of a jar,
    // and make it executable (chmod +x)

    @org.gradle.api.tasks.InputFiles
    var originalJars: ConfigurableFileTree = 
      project.fileTree("${project.buildDir}/libs") { include("*.jar") }

    @org.gradle.api.tasks.OutputDirectory
    var outputDir: File = project.buildDir.resolve("bin") // where to write the modified jar(s)

    @org.gradle.api.tasks.InputFile
    var launchScript: File = project.rootDir.resolve("launch.sh") // script to prepend

    @TaskAction
    fun createExecutableJars() {
        project.mkdir(outputDir)
        originalJars.forEach { jar ->
            outputDir.resolve(jar.name).run {
                outputStream().use { out ->
                    out.write(launchScript.readBytes())
                    out.write(jar.readBytes())
                }
                setExecutable(true)
                println("created executable: $path")
            }
        }
    }
}
```

This task extends Gradle's `DefaultTask` (Kotlin DSL), and takes three arguments:

1.  the list of "normal" jars that need to be made executable (`build/libs/*.jar` by default),
    
2.  the directory where to output the transformed jars (`bin` by default),
    
3.  the bash launch script to prepend, which needs to exist! (`<project_root>/launch.sh` by default).
    

Then, the job is straightforward: for each jar found in `inputJars`, execute the equivalent of the `cat` and `chmod` commands outlined earlier, but in Kotlin.

Note that the jars will keep the same name, so ensure the `outputDirectory` doesn't match the input directory (or the jar will be corrupted). If you don't like this, adapt the script accordingly.

### Invoking the custom task

We now need to register this new task, so it can be invoked from the command line. We also want it to run *after* the task of creating the jars. If you use the built-in `jar` task for the latter, this will do:

```kotlin
tasks.register<ExecutableJarTask>("exec-jar") {
    dependsOn("jar") // "jar" task should have run prior to it 
}
```

With this, you can now use: `bash ./gradlew exec-jar`

If you want to customize the task, for example, change the path to the launch script:

```kotlin
tasks.register<ExecutableJarTask>("exec-jar") {
    dependsOn("jar")
    // customise directly here
    launchScript = project.rootDir.resolve("bin/launcher.sh")
}
```

Done!

## An example launch script

Based on Spring Boot's script, I personally use the following launch script, which should run fine on all supported platforms:

```bash
#!/bin/sh
[[ -n "$DEBUG" ]] && set -x

# Find Java (cf: spring-boot launcher)
if [[ -n "$JAVA_HOME" ]] && [[ -x "$JAVA_HOME/bin/java" ]]; then
    javaexe="$JAVA_HOME/bin/java"
elif type -p java > /dev/null 2>&1; then
    javaexe=$(type -p java)
elif [[ -x "/usr/bin/java" ]];  then
    javaexe="/usr/bin/java"
else
    echo "Unable to find Java"
    exit 1
fi

# run jar
exec $javaexe -jar $0 "$@"
exit 0
```

Note the `exit 0` at the end: this is very important if your jar has a finite runtime (vs a Spring Boot server application). Indeed, without it, your jar will run and exit, then bash will try to execute whatever is found after the `exec` line (that is, the zipped content of the jar) and will exit with an error.

* * *

Written with ♡ by [derlin](https://github.com/derlin)