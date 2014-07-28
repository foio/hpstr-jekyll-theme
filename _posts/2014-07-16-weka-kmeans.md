---
layout: post
title: weka kmeans算法测试
description: "weka kmeans"
modified: 2014-07-16
tags: [MachineLearning  weka]
image:
  background: triangular.png
comments: true
---

weka是机器学习和数据挖掘库java库，提供有GUI界面版本和类库版本。weka是GNU通用公共许可证下发布的开源软件，这意味着我们在自己的项目自由的使用，并可以在开放源代码的情况下改进weka。在使用weka之前，我们首先要将自己的数据集转换为weka支持的格式。weka的数据格式请参考[arff格式说明][1]。学习weka库最好先有一下机器学习方面的基础知识，然后学习官方example。接下来简单说明如何使用weka库的kmeans算法。

-----

**训练数据集集**

```
@relation race

@attribute height NUMERIC
@attribute weight NUMERIC
@attribute eyeColor NUMERIC
@attribute hairColor NUMERIC

@data
170,70,1,1
180,70,2,2
175,70,1,1
200,80,2,3
210,80,2,1
200,70,2,2
150,50,1,1
155,50,1,1
170,70,1,1
173,70,1,1
170,70,1,2
```

**测试数据集集**

```
@relation race

@attribute height NUMERIC
@attribute weight NUMERIC
@attribute eyeColor NUMERIC
@attribute hairColor NUMERIC

@data
170,71,1,1
200,80,2,2
```

----


###代码

{% highlight java %}
public class KmeansTest {
	  public static void main(String[] args) throws Exception {
		    File trainFile = new File("race_train.arff");
		    File testFile  = new File("race_test.arff");
		    ArffLoader atf = new ArffLoader();  
		    // load data
		    atf.setFile(trainFile);  
		    Instances train = atf.getDataSet();
		    atf.setFile(testFile);  
		    Instances test = atf.getDataSet(); 
		    // build clusterer
		    SimpleKMeans clusterer = new SimpleKMeans();
		    clusterer.setNumClusters(2);
		    clusterer.buildClusterer(train);   
		    System.out.println(clusterer.toString());
		    // output predictions	    
		    for (int i = 0; i < test.numInstances(); i++) {
		      int cluster = clusterer.clusterInstance(test.instance(i));
		      double[] dist = clusterer.distributionForInstance(test.instance(i));
		      System.out.print((i+1));
		      System.out.print(" - ");
		      System.out.print(cluster);
		      System.out.print(" - ");
		      System.out.print(Utils.arrayToString(dist));
		      System.out.println();
		    } 
		  }
}
{% endhighlight  %}

-----

###结果如下

```
//kMeans聚类结果信息如下，比较重要的是每个聚类的中心以及误差
kMeans
======

Number of iterations: 2
Within cluster sum of squared errors: 1.7463888888888888
Missing values globally replaced with mean/mode

Cluster centroids:
                         Cluster#
Attribute    Full Data          0          1
                  (11)        (4)        (7)
============================================
height        177.5455      197.5   166.1429
weight         68.1818         75    64.2857
eyeColor        1.3636          2          1
hairColor       1.4545          2     1.1429

//分类结果如下
1 - 1 - 0.0,1.0
2 - 0 - 1.0,0.0

```

>更多资料请阅读[weka官网][2]。


  [1]: http://www.cs.waikato.ac.nz/ml/weka/arff.html
  [2]: http://www.cs.waikato.ac.nz/ml/weka/
