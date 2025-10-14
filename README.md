PS C:\Users\mshivapr022725\BFD\Bank_Fraud_Detection> java -cp target/Bank_Fraud_Detection-0.0.1-SNAPSHOT.jar src/main/java/com/example/Bank_Fraud/_Detection/weka/WekaModelTrainer.java tools/data/training_data.arff model/fraud_model.model                                                           
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:3: error: package weka.classifiers.trees does not exist
import weka.classifiers.trees.J48;
                             ^
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:4: error: package weka.core does not exist
import weka.core.Instances;
                ^
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:5: error: package weka.core.converters.ConverterUtils does not exist
import weka.core.converters.ConverterUtils.DataSource;
                                          ^
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:6: error: package weka.core does not exist
import weka.core.SerializationHelper;
                ^
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:24: error: cannot find symbol
        DataSource ds = new DataSource(arff);
        ^
  symbol:   class DataSource
  location: class WekaModelTrainer
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:24: error: cannot find symbol
        DataSource ds = new DataSource(arff);
                            ^
  symbol:   class DataSource
  location: class WekaModelTrainer
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:25: error: cannot find symbol
        Instances data = ds.getDataSet();
        ^
  symbol:   class Instances
  location: class WekaModelTrainer
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:28: error: cannot find symbol
        J48 tree = new J48();
        ^
  symbol:   class J48
  location: class WekaModelTrainer
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:28: error: cannot find symbol
        J48 tree = new J48();
                       ^
  symbol:   class J48
  location: class WekaModelTrainer
src\main\java\com\example\Bank_Fraud\_Detection\weka\WekaModelTrainer.java:37: error: cannot find symbol
        SerializationHelper.write(out, tree);
        ^
  symbol:   variable SerializationHelper
  location: class WekaModelTrainer
10 errors
error: compilation failed
