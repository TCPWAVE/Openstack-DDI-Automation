# Introduction
OpenStack is an open source cloud computing infrastructure software project. This project integrates subnet and server creation/deletion actions in Openstack with TCPWave IPAM. When a server is created or deleted or interface/floating IP is added or deleted  to a server in OpenStack, then a secure SSL rest API will be triggered to TCPWave IPAM and DNS entry  will be added or deleted based on the event. When a subnet is created or deleted in OpenStack,  similar action will take place in the IPAM. The implemented OpenStack neutron monitor will listen to the amqp events generated in neutron and sends requests to the IPAM when the events occur.

 ## Purpose
 With the OpenStack – IPAM integration, the DNS changes will happen automatically through the IPAM when servers are added/deleted, if the neutron event listener     service is configured.
 
 ## Architectural Diagram
 <img width="360" alt="openstack" src="https://user-images.githubusercontent.com/56577268/164163286-9aa6d7b3-c395-4306-ae88-529f96e71372.png">
 
 # Design
  ## Neutron Listener Service
A python script neutron_monitor.py will be created to listen to the amqp events that occur in the neutron service in Openstack when server state-related actions take place. This listener is also responsible to perform corresponding actions in the IPAM.

The communication to TCPWave IPAM from the neutron listener happens though secure SSL rest API. The appliance and client certificates must be generated and uploaded to IPAM before using them in the listener service.
The authorization of the actions in the IPAM from the listener depends entirely upon the admin that is assigned while uploading the user(client) certificate to the IPAM. This client certificate will be used to set up the authentication from neutron to the IPAM.

Current implementation supports IPv4 address space.

**Note:** The project in which servers are managed in Openstack must be present in the IPAM as an Organization entity.

When a server is created or interface is attached to the server in the OpenStack, the listener first checks if the corresponding subnet exists in the IPAM or not. If exists, it creates object with the IP address, name and mac, otherwise it creates zone level A resource record.

When a server is deleted or interface is detached from a server in the OpenStack, the listener first checks if the corresponding subnet exists in the IPAM or not. If exists, it deletes object with the IP address, otherwise it deletes zone level A resource record.

If server is created using port, the name of the port will be considered as the DNS name.

If server is created using network, the name given to the instance will be taken as the DNS name.

If a subnet is created in the OpenStack, subnet will be created in the IPAM.  The network address to which the subnet must belong in the IPAM will be taken from the tag if a tag with network CIDR is given while creating the network. Otherwise, it will be taken from the **twc_network_address** parameter value present in /etc/neutron/neutron.conf file.

If a subnet is deleted in the OpenStack, subnet will be deleted from the IPAM as well.

# Configuration

Follow the below steps to configure the neutron-listener service.

 ## Changes in /etc/neutron/neutron.conf
Add the below properties in the file /etc/neutron/neutron.conf.

In the [DEFAULT] section, add dns_domain value.

|    <br>dns_domain={DNS   zone name}              |
|--------------------------------------------------|
|    <br>Example:<br>   <br>dns_domain=test.com    |

Add [tcpwave] section with the below parameters.

|    <br>twc_address={IP Address or host name of the   TCPWave IPAM}<br>   <br>twc_client_crt={Absolute   path of the client certificate for rest API authentication to IPAM}<br>   <br>twc_client_key={Absolute   path of the client key}<br>   <br>twc_dns_zone={DNS   zone name where resource records will be added in the IPAM}<br>   <br>twc_network_address={Network   Address in the IPAM where subnets to be added}<br>   <br>twc_neutron_transport_url={Neutron transport   URL}<br>   <br>twc_openstack_identity_uri={OpenStack   Identity URI}<br>   <br>twc_openstack_network_uri=   {OpenStack Network URI}<br>   <br>     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|    <br>Example:<br>   <br>twc_address=10.1.0.24<br>   <br>twc_client_crt=/opt/tcpwave/certs/client.crt<br>   <br>twc_client_key=/opt/tcpwave/certs/client.key<br>   <br>twc_dns_zone=test.com<br>   <br>twc_network_address=20.0.0.0<br>   <br>twc_neutron_transport_url=amqp://guest:guest@10.1.0.24:5672//<br>   <br>twc_openstack_identity_uri=http://10.1.8.220:5000/v3<br>   <br>twc_openstack_network_uri=http://10.1.8.220:9696/v2.0                                                                                                                                                                                            |
  ## Configure neutron-listener service
  
  Create file /opt/tcpwave/neutron_monitor.py with the below content and give execution permissions to the file.
  
  
  



 
