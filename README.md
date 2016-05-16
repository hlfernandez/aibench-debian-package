# Generating debian packages for AIBench-based applications

This project shows how to generate debian packages for [AIBench-based](http://www.aibench.org/) application.

This is done by integrating the [jdeb](https://github.com/tcurdt/jdeb/blob/master/docs/maven.md) plugin into the default `pom.xml`.

## Usage
Building this project with `mvn package` will produce a debian package at `target/aibench-jdeb_1.0_all.deb`. This package can then be installed with `dpkg -i aibench-jdeb_1.0_all.deb` so that:
- Project files (i.e., `run.sh`, `conf`, `lib` and `plugins_bin`) are located at `/usr/local/lib/aibench-jdeb`.
- A link to launch script is located at `/usr/local/bin/run-aibench`. 

This way, the application can be run by running command `run-aibench` from any path.

## Step-by-step guide

### 1 Create an AIBench project

First, we create a new AIBench project:

```bash
mvn archetype:generate -DarchetypeGroupId=es.uvigo.ei.sing -DarchetypeArtifactId=aibench-archetype -DarchetypeVersion=2.5.5  -DgroupId=es.uvigo.ei.sing -DartifactId=my-aibench-application -DinteractiveMode=false -DarchetypeCatalog=http://sing.ei.uvigo.es/maven2/archetype-catalog.xml
```

### 2 Configure pom.xml

Secondly, we configure the `pom.xml` so that we:
- Set a proper project name and add a description.
- Add property `debian.package.lib.folder` specifying the location of the project in target debian systems.
- Add property `debian.package.command` specifying the location of the command in target debian systems.
- Add property `debian.package.maintainer` specifying the project maintainer.

### 3 Create necessary resource files

At this momment, we have to create two files:
- An adapted `run.sh` for the target system. This file will be `src/main/resources/run-debian-package.sh`.
- A debian control file at `src/main/resources/debian/control/control`.

#### 3.1 `src/main/resources/run-debian-package.sh`

This script is pretty similar to the default `run.sh`:
```bash
#!/bin/bash
cd ${debian.package.lib.folder}
java -jar ./lib/aibench-aibench-2.5.5.jar
```

#### 3.2 `src/main/resources/debian/control/control`

When creating this file we use properties defined in project's `pom.xml`:
```bash
Package: ${artifactId}
Version: ${version}
Section: misc
Priority: low
Architecture: all
Description: ${description}
Maintainer: ${debian.package.maintainer}
```

#### 3.3 Copy new resources

It is necessary to add this `execution` to the `maven-resources-plugin` so that new resources created in the previous step are copied (and filtered) to the project build directory:

```bash
<!-- copy debian files -->
<execution>
	<id>copy-resources-debian</id>
	<phase>validate</phase>
	<goals>
		<goal>copy-resources</goal>
	</goals>
	<configuration>
		<outputDirectory>${project.build.directory}/debian</outputDirectory>
		<resources>
			<resource>
				<directory>src/main/resources/debian</directory>
				<filtering>true</filtering>
			</resource>
	
		</resources>
	</configuration>
</execution>
```
### 4 Adding `jdeb` plugin

Finally, we add the following `jdeb` plugin definition to project `pom.xml`:
```bash
<plugin>
	<artifactId>jdeb</artifactId>
	<groupId>org.vafer</groupId>
	<version>1.5</version>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>jdeb</goal>
			</goals>
			<configuration>
				<controlDir>${project.build.directory}/debian/control</controlDir>
				<dataSet>
					<data>
						<src>${project.build.directory}/run-debian-package.sh</src>
						<dst>run.sh</dst>
						<type>file</type>
						<mapper>
						<type>perm</type>
							<prefix>${debian.package.lib.folder}</prefix>
							<filemode>744</filemode>
						</mapper>
					</data>
					<data>
						<src>${project.build.directory}/conf</src>
						<type>directory</type>
						<mapper>
							<type>perm</type>
							<prefix>${debian.package.lib.folder}/conf</prefix>
						</mapper>
					</data>
					<data>
						<src>${project.build.directory}/lib</src>
						<type>directory</type>
						<mapper>
							<type>perm</type>
							<prefix>${debian.package.lib.folder}/lib</prefix>
						</mapper>
					</data>
					<data>
						<src>${project.build.directory}/plugins_bin</src>
						<type>directory</type>
						<mapper>
							<type>perm</type>
							<prefix>${debian.package.lib.folder}/plugins_bin</prefix>
						</mapper>
					</data>
					<data>
						<type>link</type>
						<linkName>${debian.package.command}</linkName>
						<linkTarget>${debian.package.lib.folder}/run.sh</linkTarget>
						<symlink>true</symlink>
					</data>
				</dataSet>
			</configuration>
		</execution>
	</executions>
</plugin>
```