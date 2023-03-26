---
layout: post
current: post
cover: assets/built/images/python/jpype.webp
navigation: True
title: jaydebeapi - Class not found Error
date: 2023-03-25 22:30:00 +0900
tags: [python]
class: post-template
subclass: 'post tag-python'
author: GyuhoonK
---

jaydebeapi로 DB 연결 시 `TypeError: Class not found` 원인 파악

### 에러 로그

Oracle와의 `connector` 생성을 위해 아래와 같이 코드를 작성하여 실행했습니다. 

```python
conn = jaydebeapi.connect(jclassname="oracle.jdbc.driver.OracleDriver",
                          url="jdbc:oracle://host:port/",
                          jars = "ojdb6.jar")
```

실행 시 아래와 같이 `jclassname`을 찾을 수 없다는 내용의 에러가 발생했습니다(Class not found).

```
Traceback (most recent call last):
  File "connector.py", line 46, in <module>
    conn = jaydebeapi.connect(
  File "/usr/bin/python3/site-packages/jaydebeapi/__init__.py", line 412, in connect
    jconn = _jdbc_connect(jclassname, url, driver_args, jars, libs)
  File "/usr/bin/python3/site-packages/jaydebeapi/__init__.py", line 221, in _jdbc_connect_jpype
    jpype.JClass(jclassname)
  File "/usr/bin/python3/site-packages/jpype/_jclass.py", line 99, in __new__
    return _jpype._getClass(jc)
TypeError: Class oracle.jdbc.driver.OracleDriver is not found
```

### 원인 파악을 위한 trial and error

- jar file  
`jclassname`, 즉 실행할 클래스를 java 경로에서 찾지 못했다는 이야기이므로 `jars`로 전달한 `ojdb6.jar`을 의심했습니다. 1) jar 파일 내에 클래스가 존재하지 않거나, 2) 제가 jars를 입력하는 과정에서 오타가 있었을 수 있다고 생각했습니다. 
먼저, jar 파일 내에 클래스가 존재하는지 확인했습니다.

```bash
$ jar tf ojdb6.jar | grep OracleDriver
oracle/jdbc/driver/OracleDriver.class
```

아쉽게도(?) jar파일 내에 `oracle.jdbc.driver.OracleDriver` 클래스가 존재하고 있었습니다. 또한 제가 jars파일 이름(`ojdb6.jar`)을 입력하는 과정에서 문제가 있었던 것도 아니었습니다.

- java path / jdk version  
Java 버전의 의존성 문제일 수도 있다고 생각했습니다. 현재 M1 칩셋의 맥북을 사용 중이라서 아래와 같이 Zulu java를 사용 중입니다. `JAVA_HOME`도 정상적으로 설정되어 있었습니다.  

```bash
$ java --version
openjdk 15.0.10 2023-01-17
OpenJDK Runtime Environment Zulu15.46+17-CA (build 15.0.10+5-MTS)
OpenJDK 64-Bit Server VM Zulu15.46+17-CA (build 15.0.10+5-MTS, mixed mode)
$ echo $JAVA_HOME
/Users/user/zulu15.46.17-ca-jdk15.0.10-macosx_aarch64/zulu-15.jdk/Contents/Home
```

jar 파일을 압축해제하여  `META-INF/MANIFEST.MF` 내용을 확인할 수 있습니다. odjbc는 oracle 홈페이지에서 직접 버전 정보를 확인할 수 있어서 불필요하긴 합니다. hive jdbc로 예를 들어보겠습니다. cloudera에서 배포한 HiveJDBC42의 빌드에서 사용된 jdk 버전은 1.8이었습니다.
```bash
$ jar xvf HiveJDBC42-2.6.11.1014.jar # jar 파일 압축 해제
$ cat META-INF/MANIFEST.MF 
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: SYSTEM
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_111
```

현재 설치된 jdk는 openjdk15이고, 빌드를 jdk1.8로 했으니 java의 문제는 아닌 것으로 보입니다.

### 진찌 원인: JVM
셋팅의 문제는 아니라는 걸 파악했고, 이번에는 `jaydebeapi` 내부 코드를 살펴보았습니다. jaydebeapi는 내부에서 `jpype` 라이브러리를 이용하여 JVM을 실행하고, JVM 내에서 전달받은 jar파일의 클래스를 실행합니다. 

아래 코드는 `jaydebeapi`에서 JVM 셋팅(classpath, librarypath, driver_args)을 하고, JVM을 시작합니다. 그런데, `if not jpype.isJVMStarted()` 조건 때문에 이미 실행 중인 JVM이 있는 경우에는 동작하지 않는 코드입니다. 실행되고 있는 JVM이 없는 경우에만 connect 객체 생성을 위해 전달한 `jars`가 JVM의 `Djava.libaray.path`에 포함될 수 있는 구조입니다.

```python
def _jdbc_connect_jpype(jclassname, url, driver_args, jars, libs):
    import jpype
    if not jpype.isJVMStarted():
        args = []
        class_path = []
        if jars:
            class_path.extend(jars)
        class_path.extend(_get_classpath())
        if class_path:
            args.append('-Djava.class.path=%s' %
                        os.path.pathsep.join(class_path))
        if libs:
            # path to shared libraries
            libs_path = os.path.pathsep.join(libs)
            args.append('-Djava.library.path=%s' % libs_path)
        # jvm_path = ('/usr/lib/jvm/java-6-openjdk'
        #             '/jre/lib/i386/client/libjvm.so')
        jvm_path = jpype.getDefaultJVMPath()
        global old_jpype
        if hasattr(jpype, '__version__'):
            try:
                ver_match = re.match('\d+\.\d+', jpype.__version__)
                if ver_match:
                    jpype_ver = float(ver_match.group(0))
                    if jpype_ver < 0.7:
                        old_jpype = True
            except ValueError:
                pass
        if old_jpype:
            jpype.startJVM(jvm_path, *args)
        else:
            jpype.startJVM(jvm_path, *args, ignoreUnrecognized=True,
                           convertStrings=True)
```

따라서, jpype에 의해 이미 실행 중인 JVM이 존재한다면, oracle connector를 생성하기 위해 전달한 jar 파일은 어디에도 사용되지 않습니다. 프로그래머는 변수를 입력했지만, 코드 내부에서는 해당 변수가 어떤 곳에서도 사용되지 않습니다. 이런 경우에 JVM에 해당 jar 파일이 없으므로, Class not found가 발생하는 것도 이해가 됩니다.

실제로, 제가 작성 중인 코드에서 oracle connector를 생성하기 전에 mysql db와 연결하기 위한 다른 connector를 이미 생성해놓고 있었습니다. 따라서 아래와 같은 이유로 Class not found 에러가 발생했습니다.

```python
import jpype
conn = jaydebeapi.connect(jclassname="com.mysql.jdbc.Driver",
                          url="jdbc:mysql://host:port/",
                          jars = "mysql-connector-java-5.1.45.jar")

print(f"is JVM already started? {jpype.isJVMStarted()}")
# 출력 결과: is JVM already started? True
# 이 상태에서 실행 중인 JVM의 Djava.class.path(CLASSPATH)에는 mysql-connector-java-5.1.45.jar만 존재
...
conn = jaydebeapi.connect(jclassname="oracle.jdbc.driver.OracleDriver",
                          url="jdbc:oracle://host:port/",
                          jars = "ojdb6.jar")
# 이미 JVM이 실행 중이므로 jpype로 JVM을 시작하는 작업을 수행하지 않는다.
# 따라서 JVM의 CLASSPATH에는 mysql-connector-java-5.1.45.jar만 존재하므로, OracleDriver 클래스를 찾을 수 없다.
```

[jaydebeapi github](https://github.com/baztian/jaydebeapi/issues/85#issuecomment-442851010)를 들어가서 살펴보니 저와 동일한 현상에 대한 언급이 있었습니다.

>Also, if you are pulling from multiple database connections, ie. making multiple jaydebeapi.connect() calls, the jars parameter in your first connection call must contain all of the paths to your jdbc jar files. Any jar parameter in subsequent connect() calls seems to be ignored.

- 그냥 JVM을 shutdown했다가 재시작하면 안될까?  
jpype는 JVM을 종료(shutdown)하는 API도 제공합니다.
```python
jpype.startJVM()
jpype.shutdownJVM()
```

그렇다면 두번째 connector를 생성하기 전에 JVM을 종료하면 jar 파일을 다시 업로드할 수 있지 않을까 생각했습니다. 

```python
connector1 = jaydebeapi.connect(args1)
jpype.shutdownJVM()
connector2 = jaydebeapi.connect(args2)
```

하지만, 아쉽게도 jpype는 JVM을 shutdown하면 재가동할 수 없습니다.

>This method shuts down the JVM and disables access to existing Java objects. Due to limitations in the JPype, it is not possible to restart the JVM after being terminated.

따라서 위와 같이 실행 시에는 에러(`OSError: JVM cannot be restarted`)가 발생합니다. 
```
OSError: JVM cannot be restarted
```

### 해결책
따라서 위 코드가 동작하려면 아래와 같이 첫번째 connect 생성 시에 모든 jar 파일을 추가해주어야합니다. 
```python
conn = jaydebeapi.connect(jclassname="com.mysql.jdbc.Driver",
                        url="jdbc:mysql://host:port/",
                        jars = ["mysql-connector-java-5.1.45.jar", "ojdb6.jar"])

print(f"is JVM already started? {jpype.isJVMStarted()}")
# 출력 결과: is JVM already started? True
# 이 상태에서 실행 중인 JVM의 Djava.libaray.path(CLASSPATH)에는 mysql-connector-java-5.1.45.jar와 ojdb6.jar가 추가됨
...
conn = jaydebeapi.connect(jclassname="oracle.jdbc.driver.OracleDriver",
                        url="jdbc:oracle://host:port/",
                        jars = "ojdb6.jar" # 이미 JVM에 해당 jar가 추가되었고, 두번째 connect()부터는 해당 옵션은 무시되므로 추가하지 않아도 된다
                        )
# 이미 실행되고 있는 JVM에서 connector를 생성한다
```


[참고]  
[jaydebeapi github issues#85](https://github.com/baztian/jaydebeapi/issues/85#issuecomment-442851010)  
[jpype Document](https://jpype.readthedocs.io/en/latest/api.html)