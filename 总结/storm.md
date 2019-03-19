
storm 官方docker文档建立

* docker run -d --restart always --name some-zookeeper zookeeper

* docker run -d --restart always --name some-nimbus --link some-zookeeper:zookeeper storm storm nimbus

* docker run -d --restart always --name supervisor --link some-zookeeper:zookeeper --link some-nimbus:nimbus storm storm supervisor


* docker run -d -p 8080:8080 --restart always --name ui --link some-nimbus:nimbus storm storm ui

* docker run --link some-nimbus:nimbus -it --rm -v $(pwd)/StormDemo.jar:/topology.jar storm storm jar /topology.jar com.maxplus1.DemoApplication topology

* docker run -it -v $(pwd)/StormDemo.jar:/topology.jar storm storm jar /topology.jar com.maxplus1.DemoApplication


 

[kafka docker](compose.yml)
