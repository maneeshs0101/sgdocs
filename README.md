PS C:\Users\mshivapr022725\BFD\Bank_Fraud_Detection> mvn -q compile org.codehaus.mojo:exec-maven-plugin:3.1.0:java `
>>   -Dexec.mainClass="com.example.Bank_Fraud._Detection.weka.WekaModelTrainer" `                                                                   
>>   -Dexec.args="tools/data/training_data.arff model/fraud_model.model"                                                                            
WARNING: A restricted method in java.lang.System has been called
WARNING: java.lang.System::load has been called by org.fusesource.jansi.internal.JansiLoader in an unnamed module (file:/C:/Users/mshivapr022725/bin/apache-maven-3.9.9/lib/jansi-2.4.1.jar)
WARNING: Use --enable-native-access=ALL-UNNAMED to avoid a warning for callers in this module
WARNING: Restricted methods will be blocked in a future release unless native access is enabled

WARNING: A terminally deprecated method in sun.misc.Unsafe has been called
WARNING: sun.misc.Unsafe::objectFieldOffset has been called by com.google.common.util.concurrent.AbstractFuture$UnsafeAtomicHelper (file:/C:/Users/mshivapr022725/bin/apache-maven-3.9.9/lib/guava-33.2.1-jre.jar)
WARNING: Please consider reporting this to the maintainers of class com.google.common.util.concurrent.AbstractFuture$UnsafeAtomicHelper
WARNING: sun.misc.Unsafe::objectFieldOffset will be removed in a future release
[ERROR] [ERROR] Some problems were encountered while processing the POMs:
[FATAL] Non-resolvable parent POM for com.example:Bank_Fraud_Detection:0.0.1-SNAPSHOT: The following artifacts could not be resolved: org.springframework.boot:spring-boot-starter-parent:pom:3.5.6 (present, but unavailable): org.springframework.boot:spring-boot-starter-parent:pom:3.5.6 was not found in https://sofa.dns20.socgen/nexus/repository/maven-group/ during a previous attempt. This failure was cached in the local repository and resolution is not reattempted until the update interval of thirdparty-releases has elapsed or updates are forced and 'parent.relativePath' points at no local POM @ line 5, column 10
 @ 
[ERROR] The build could not read 1 project -> [Help 1]
[ERROR]   
[ERROR]   The project com.example:Bank_Fraud_Detection:0.0.1-SNAPSHOT (C:\Users\mshivapr022725\BFD\Bank_Fraud_Detection\pom.xml) has 1 error
[ERROR]     Non-resolvable parent POM for com.example:Bank_Fraud_Detection:0.0.1-SNAPSHOT: The following artifacts could not be resolved: org.springframework.boot:spring-boot-starter-parent:pom:3.5.6 (present, but unavailable): org.springframework.boot:spring-boot-starter-parent:pom:3.5.6 was not found in https://sofa.dns20.socgen/nexus/repository/maven-group/ during a previous attempt. This failure was cached in the local repository and resolution is not reattempted until the update interval of thirdparty-releases has elapsed or updates are forced and 'parent.relativePath' points at no local POM @ line 5, column 10 -> [Help 2]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/ProjectBuildingException
[ERROR] [Help 2] http://cwiki.apache.org/confluence/display/MAVEN/UnresolvableModelException
PS C:\Users\mshivapr022725\BFD\Bank_Fraud_Detection> 
