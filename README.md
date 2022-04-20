# Introduction
OpenStack is an open source cloud computing infrastructure software project. This project integrates subnet and server creation/deletion actions in Openstack with TCPWave IPAM. When a server is created or deleted or interface/floating IP is added or deleted  to a server in OpenStack, then a secure SSL rest API will be triggered to TCPWave IPAM and DNS entry  will be added or deleted based on the event. When a subnet is created or deleted in OpenStack,  similar action will take place in the IPAM. The implemented OpenStack neutron monitor will listen to the amqp events generated in neutron and sends requests to the IPAM when the events occur.

 ## Purpose
 With the OpenStack – IPAM integration, the DNS changes will happen automatically through the IPAM when servers are added/deleted, if the neutron event listener     service is configured.
 
 ## Architectural Diagram
 
