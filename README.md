#Docker Container Network Bandwidth Management

Docker technology makes it easier to contain an application and makes it easier to deploy and run containers either in data center or in a Cloud. Native Docker run command has various options to manage container resources like CPU, Memory, Disk but there is no option to manage network resource utilization.
Effective management of network resources will lead to better utilization of underlying hardware and better performance isolation. Docker has no option to manage the network bandwidth based on per container at present.

#Proposed Solution and use cases

So the proposed Docker Container Network Bandwidth Management solution provides the means to manage Docker host networking resources. It allows the network traffic to be allocated or deallocate based on container id. Also users can prioritize the crucial traffic by limiting the usage of network bandwidth of less important container, so, this could helps making the host network resources usage  in predictable manner and overall expected containers network throughput behavior.

The approach is to set a specific classid in the net_cls cgroup to the docker container and make use of traffic classifier(tc) tool to shape the network traffic of the container based on its classid. This feature make use of the physical network interface card speed and allocation of network bandwidth to the containers beyond actual speed will be restricted. Current implementation works by applying the bandwidth limiting rules using TC(HTB) for both incoming and outgoing traffic by getting the public facing port number of a container(HostPort).


So adding a new option to the $docker network command to manage the network traffic dynamically just by using the running container Id.
Managing the container network resources by docker itself would benefit docker SWARM(helpful in scheduling the containers based on the available Network IO too), AWS or any other Cloud service providers who offers docker containers.  Also this should be a good option to limit the network resource usage of a container dynamically



Example illustration shown in the below diagram:

![Host Bandwidth logo](https://github.com/shivacherukuri/Docker-Network-Bandwidth/blob/master/docs/static_files/Docker-bwimg-v27pub.png "Host Bandwidth")


#New Command usage and sample example

The new command option usage as follows:

          Usage:  docker network bandwidth CONTAINER [OPTIONS]
          
          Set/Remove Network Bandwidth Management rules
          
          Aliases:
            bandwidth, bandwidth
          
          Options:
            -i, --InterfaceName string   Docker Host Physical Interface Name(optional in the case of bridge)
            -r, --Remove                 Remove Container Network Bandwidth
            -s, --Set value              Set Container Network Bandwidth RATE
            -t, --SpeedTypeIn string     SpeedType [kbps|mbps|gbps] bits per second
                --help                   Print usage



Example Usage of the new feature by using any docker OS image with iperf binary:

Continue it by starting two or more containers in the following way which has iperf binary

Start Container 1

            $docker run -ti --net=bridge -p=22222:8880  --name iperf_server21 07c6fa6bd4c5 /bin/sh
            
Start Container 2

            $docker run -ti --net=bridge -p=33333:9090  --name iperf_server33  07c6fa6bd4c5 /bin/sh
            

Let's see docker ps command output

  
           $docker ps
          CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
          2e66a538f3b0        07c6fa6bd4c5        "/bin/sh"           10 minutes ago      Up 10 minutes       0.0.0.0:33333->9090/tcp   iperf_server33
          a722daa03c7b        07c6fa6bd4c5        "/bin/sh"           12 minutes ago      Up 12 minutes       0.0.0.0:22222->8880/tcp   iperf_server21


Two containers are up and running and to test the bandwidth throttling feature, using iperf tool running inside containers 

Start iperf server inside the two container's

container 1 :

          $iperf -s -p 8880 -i 10
container 2 :

          $iperf -s -p 9090 -i 10

This makes the 'iperf' server listening for connection from the remote client.
Now send traffic from the remote clients to both the iperf servers running inside containers by using the 
containers public facing port number, in this example container #1 mapped to port number 22222 and container #2 mapped to 33333 port number

Remote client 1 runs: 

          $iperf -c "docker container 1 host ip" -d -p 22222 -t 200 -i 10
Remote client 2 runs: 

          $iperf -c "docker container 2 host ip" -d -p 33333 -t 200 -i 10


Now you can observe the network throughput output from iperf server in each container terminal. If your host default network physical NIC speed is 10Gig, then you may see two containers sharing the bandwidth around 5gbps each. Suppose if container #2 needs more bandwidth than container #1 then you can apply the new bandwidth rule to container #1 network bandwidth usage.


To set the bandwidth 100 kbps(kilo bits per second) to container #1, use the below command

          $docker network bandwidth a722daa03c7b -s 100 -t kbps


#Demo

Please view the ![demo video](https://github.com/shivacherukuri/Docker-Network-Bandwidth/blob/master/demo/docker-BW-demo2.mp4)  to see how the new docker network bandwidth feature really works. Also please try more experiments and let me know the feedback.

For the code diff of this new feature you can check the branch:   
<a href="https://github.com/shivacherukuri/docker"> siva-nw-bandwidth branch </a>.

#Feedback

If you have any further queries or feedback please feel free to email <a href="mailto:sivaramaprasad.c@hpe.com">me</a> or

 at the proposal #27809 ![27809](https://github.com/docker/docker/issues/27809)


#Copyright

          (c) Copyright 2016 Hewlett Packard Enterprise Development LP
