# QoS / DiffServ (SPCOR Ch 21)

in QoS you have to do the following:
- Mark the packet EF, AF and so on
- Policy, Shape or prioritise the packet
There are different queueing mechanisms used to prioritise the packet, depending on the use case, you need to use one over another

## QoS Models ##
DiffServ - mostly used in today's network
- designed to overcome the limitations of Best Effort and Intserv models
- Network traffic is identified by classes (class-maps)
- Network QoS policy enforce differentiated treatment of traffic classes
- You choose the level of service for each class 

Class-map 1
    voip traffic priority 1 (more prioritise)
class-map 2
    bulk data priority 2 
class-map 3
    everything else priority 3 (less preffered - drop or buffer if network is congested)

**Modular QoS CLI Components**
1 - Class Map: define classes of traffic (What traffic do we care about?)
    - You can use access-list to match the traffic
    - match source interface too 
2 - Policy Map: Define QoS policies for classes (what will be done to this traffic?)
3 - Service Policy - Apply a service policy (Where will this policy be implemented?) - attach to an interface

