# Introduction
OpenStack is an open source cloud computing infrastructure software project. This project integrates subnet and server creation/deletion actions in Openstack with TCPWave IPAM. When a server is created or deleted or interface/floating IP is added or deleted  to a server in OpenStack, then a secure SSL rest API will be triggered to TCPWave IPAM and DNS entry  will be added or deleted based on the event. When a subnet is created or deleted in OpenStack,  similar action will take place in the IPAM. The implemented OpenStack neutron monitor will listen to the amqp events generated in neutron and sends requests to the IPAM when the events occur.

 ## Purpose
 With the OpenStack – IPAM integration, the DNS changes will happen automatically through the IPAM when servers are added/deleted, if the neutron event listener     service is configured.
 
 ## Architectural Diagram
 ![openstack](https://user-images.githubusercontent.com/56577268/164609519-e8c8a3d3-b65b-4a8c-b442-bc5666425d49.png)
 
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

    dns_domain={DNS zone name}
    Example:
    dns_domain=test.com

Add **[tcpwave]** section with the below parameters.

    twc_address={IP Address or host name of the TCPWave IPAM}
    twc_client_crt={Absolute path of the client certificate for rest API authentication to IPAM}
    twc_client_key={Absolute path of the client key}
    twc_dns_zone={DNS zone name where resource records will be added in the IPAM}
    twc_network_address={Network Address in the IPAM where subnets to be added}
    twc_neutron_transport_url={Neutron transport URL}
    twc_openstack_identity_uri={OpenStack Identity URI}
    twc_openstack_network_uri= {OpenStack Network URI}
    
    Example:
    twc_address=10.1.0.24
    twc_client_crt=/opt/tcpwave/certs/client.crt
    twc_client_key=/opt/tcpwave/certs/client.key
    twc_dns_zone=test.com
    twc_network_address=20.0.0.0
    twc_neutron_transport_url=amqp://guest:guest@10.1.0.24:5672//
    twc_openstack_identity_uri=http://10.1.8.220:5000/v3
    twc_openstack_network_uri=http://10.1.8.220:9696/v2.0

  ## Configure neutron-listener service
  
  Create file /opt/tcpwave/neutron_monitor.py with the below content and give execution permissions to the file.
  
    #!/usr/bin/env python
 
    # Copyright 2022 TCPWave Inc.
    # All Rights Reserved.
    #
    # TCPWave neutron monitor listens to the amqp messages of neutron service and performs actions based on the events. 
    # Object will be added in the TCPWave IPAM when the monitor sees the event port.update.start.
    # Object will deleted from the TCPWave IPAM when the monitor sees the events port.update.end or port.delete.
    # 
    # If there is a subnet present in the IPAM for the IP address in the specified organization in /etc/neutron/neutron.conf, 
    # then object will be added/deleted to the subnet in the IPAM.
    # Otherwise resource record will be added/deleted at zone level.

 
    import string
    import ipaddress
    import netaddr
    import datetime
    import sys
    import json
    import logging as log
    import urllib.request
    import urllib.error
    import ssl
 
    from configparser import ConfigParser
     
    from kombu import BrokerConnection
    from kombu import Exchange
    from kombu import Queue
    from kombu.mixins import ConsumerMixin
    from oslo_config import cfg
    from oslo_service import service
    import oslo_messaging
    import requests
    import ipaddress
     
    tcpwave_group = cfg.OptGroup(name='tcpwave',title='Tcpwave Group')
    TWC_configFileName="/etc/neutron/neutron.conf"
     
    log.basicConfig(filename='/var/log/neutron/server.log', level="DEBUG", format='%(asctime)s %(message)s')
     
    tcpwave_twc_parameters = [
    cfg.StrOpt('twc_neutron_transport_url', default=None, help=("TCPWave Neutron Monitor Transport URL")), 
    cfg.StrOpt('twc_address', default=None, help=("TWC IP Address")),
    cfg.StrOpt('twc_client_crt', default=None, help=("Client certificate path")),
    cfg.StrOpt('twc_client_key', default=None, help=("Client cerificate key path")),
    cfg.StrOpt('twc_dns_zone', default=None, help=("TWC DNS Zone")),
    cfg.StrOpt('twc_network_address', default=None, help=("TWC Network Address")),
    cfg.StrOpt('twc_openstack_identity_uri', default=None, help=("OpenStack Identity URI")),
    cfg.StrOpt('twc_openstack_network_uri', default=None, help=("OpenStack Network URI"))]
     
    version = 1.0
     
    EXCHANGE_NAME="neutron"
    ROUTING_KEY="notifications.info"
    QUEUE_NAME="tcpwave_neutron_monitor"
    FLOAT_START="floatingip.create.start"
    FLOAT_END="floatingip.create.end"
    FLOAT_U_START="floatingip.update.start"
    FLOAT_U_END="floatingip.update.end"
    PORT_START="port.create.start"
    PORT_END="port.create.end"
    PORT_DELETE_END="port.delete.end"
    PORT_U_START="port.update.start"
    PORT_U_END="port.update.end"
     
    SUBNET_CREATE_START="subnet.create.start"
    SUBNET_CREATE_END="subnet.create.end"
    SUBNET_DELETE_END="subnet.delete.end"
    SUBNET_UPDATE_END="subnet.update.end"
     
    NETWORK_CREATE_START="network.create.start"
    NETWORK_CREATE_END="network.create.end"
    NETWORK_DELETE_END="network.delete.end"
    NETWORK_UPDATE_END="network.update.end"
 
 
    def config_parser(conf,list):
        CONF = cfg.CONF
        CONF.register_group(tcpwave_group)
        CONF.register_opts(list, "tcpwave")
        CONF(default_config_files=conf)
        return CONF
 
     NEUTRON_CONF=config_parser(['/etc/neutron/neutron.conf'],tcpwave_twc_parameters)
     monitor_broker = NEUTRON_CONF.tcpwave.twc_neutron_transport_url
     log.debug('TCPWave Neutron Monitor Transport URL = '+monitor_broker)
      
     def getTWCConfig(configFileName):
         TWC_CONF=config_parser([configFileName],tcpwave_twc_parameters)
      
         db = {}
         db['twc_address']=TWC_CONF.tcpwave.twc_address
         db['twc_client_crt']=TWC_CONF.tcpwave.twc_client_crt
         db['twc_client_key']=TWC_CONF.tcpwave.twc_client_key
         db['twc_dns_zone']=TWC_CONF.tcpwave.twc_dns_zone    
         db['twc_network_address']=TWC_CONF.tcpwave.twc_network_address
      db['twc_openstack_identity_uri']=TWC_CONF.tcpwave.twc_openstack_identity_uri
      db['twc_openstack_network_uri']=TWC_CONF.tcpwave.twc_openstack_network_uri   
      
   
      return db
   
      # Returns address type 4 or 6
      def enumIPtype(address):
          address = ipaddress.ip_address(unicode(address))
          return address.version
       
      # Splits FQDN into host and domain portions
      def splitFQDN(name):
          hostname = name.split('.')[0]
          domain = name.partition('.')[2]
          return hostname,domain
       
      def addV4Object(ip, hostname, mac, project_id):
          deleteV4Object(ip, hostname, project_id)
          paramsTWC = getTWCConfig(TWC_configFileName)
          ctx = ssl.create_default_context()
          ctx.check_hostname = False
          ctx.verify_mode = ssl.CERT_NONE
          ctx.load_cert_chain(paramsTWC['twc_client_crt'], paramsTWC['twc_client_key']);
          host = paramsTWC['twc_address']
          org = getProjectName(project_id)
          zone = paramsTWC['twc_dns_zone']
           
          log.debug('TCPWave neutron listener: Org = '+org)
          log.debug('TCPWave neutron listener: Zone = '+zone)    
          log.debug('TCPWave neutron listener: IP address = '+ip)
          log.debug('TCPWave neutron listener: host name = '+hostname)
          log.debug('TCPWave neutron listener: mac address = '+mac)
          
          #Get subnet of the IP Address
          url = 'https://'+host+':7443/tims/rest/object/getSubnetByObjectIP?address='+ip+'&organization='+org
          req = urllib.request.Request(url=url, method='GET')
          cidr = ''
          try:
             f  = urllib.request.urlopen(req, context=ctx)
             log.debug(str(f.status)+' '+f.reason)  
             data = json. load(f)
             cidr = data['fullAddress']+'/'+str(data['mask_length'])
          except urllib.error.HTTPError as e:
              log.debug (e.code)
              log.debug (e.reason)
              log.debug (e.read())
       
          if(cidr == ''):
              log.debug('TCPWave neutron listener: adding zone level RR in the IPAM.')
              url = 'https://'+host+':7443/tims/rest/zone/rr/add'
              d = {   'organization_name': org,
                      'rrtype': 'A',
                      'rrclass': 'IN',
                      'zoneName': zone,
                      'owner': hostname,
                      'data': ip,
                      'ttl':1200,
                      'description':'Added by Openstack neutron listener.'
                  }
              try:
                  data = json.dumps(d)
                  req = urllib.request.Request(url=url, data=data.encode(), method='POST')
            req.add_header('Content-Type', 'application/json')
            f  = urllib.request.urlopen(req, context=ctx)
            log.debug('TCPWave neutron listener: response')
            log.debug(f.status, f.reason)
            log.debug(f.read())
            f.close()
            log.debug('TCPWave neutron listener: adding zone level RR in the IPAM is completed successfully.')
        except urllib.error.HTTPError as e:
            log.debug('TCPWave neutron listener: error occured while adding zone level RR.')
            log.debug (e.code)
            log.debug (e.reason)
            log.debug (e.read())
           else:
               #create Object
               log.debug('TCPWave neutron listener: adding object in the IPAM.')
               url = 'https://'+host+':7443/tims/rest/object/add'
               ip_bits = str(ip).split('.')
               d = { 'organization_name': org,
                           'subnet_address': cidr,
                           'addr1': str(ip_bits[0]),
                           'addr2': str(ip_bits[1]),
                           'addr3': str(ip_bits[2]),
                           'addr4': str(ip_bits[3]),
                           'name': hostname,
                           'domain_name': zone,
                           'class_code':'Others',
                           'alloc_type':'1',
                           'update_ns_a': True,
                           'update_ns_ptr': True,
                           'dyn_update_rrs_a': True,
                           'dyn_update_rrs_ptr': True,
                           'dyn_update_rrs_cname':  True,
                           'dyn_update_rrs_mx': True,
                           'mac':mac,
                           'description':'Added by Openstack neutron listener.'}
               try:
                   data = json.dumps(d)
                   req = urllib.request.Request(url=url, data=data.encode(), method='POST')
                   req.add_header('Content-Type', 'application/json')
                   f  = urllib.request.urlopen(req, context=ctx)
                   log.debug('TCPWave neutron listener: response')
                   log.debug(f.status, f.reason)
                   log.debug(f.read())
                   f.close()
                   log.debug('TCPWave neutron listener: adding object in the IPAM is completed successfully.')
               except urllib.error.HTTPError as e:
                   log.debug('TCPWave neutron listener: error occured while adding object.')
                   log.debug (e.code)
                   log.debug (e.reason)
                   log.debug (e.read())
                   
               def deleteV4Object(ip, hostname, project_id):
                   if(hostname==''):
                     log.debug('TCPWave neutron listener: Hostname is empty. So cannot delete the resource record')
                     return;
                   paramsTWC = getTWCConfig(TWC_configFileName)
                   ctx = ssl.create_default_context()
                   ctx.check_hostname = False
                   ctx.verify_mode = ssl.CERT_NONE
                   ctx.load_cert_chain(paramsTWC['twc_client_crt'], paramsTWC['twc_client_key']);
                   host = paramsTWC['twc_address']
                   org = getProjectName(project_id)
                   zone = paramsTWC['twc_dns_zone']
                   owner = hostname+'.'+zone+'.'
                   
                   log.debug('TCPWave neutron listener: Zone = '+zone)
                   log.debug('TCPWave neutron listener: Org = '+org)
                   log.debug('TCPWave neutron listener: IP address = '+ip)
                
                
                   #Get subnet of the IP Address
                   url = 'https://'+host+':7443/tims/rest/object/getSubnetByObjectIP?address='+ip+'&organization='+org
                   req = urllib.request.Request(url=url, method='GET')
                   cidr = ''
                   subnet_id = ''
                   try:
                      f  = urllib.request.urlopen(req, context=ctx)
                      log.debug(str(f.status)+' '+f.reason)  
                      data = json. load(f)
                      cidr = data['fullAddress']+'/'+str(data['mask_length'])
                      subnet_id = str(data['id'])
                   except urllib.error.HTTPError as e:
                       log.debug (e.code)
                       log.debug (e.reason)
                       log.debug (e.read())
                
                   if(cidr == ''):
                       log.debug('TCPWave neutron listener: deleting zone level RR in the IPAM.')
                       url = 'https://'+host+':7443/tims/rest/zone/rr/delete'
                       d = { 'organization_name': org,
                                   'rrtype': 'A',
                                   'rrclass': 'IN',
                                   'zoneName': zone,
                                   'owner': owner,
                                   'data': ip,
                                   'proxy':0
                           }
                       try:
                           data = json.dumps(d)
                           req = urllib.request.Request(url=url, data=data.encode(), method='POST')
                           req.add_header('Content-Type', 'application/json')
                           f  = urllib.request.urlopen(req, context=ctx)
                           log.debug(f.status, f.reason)
                           log.debug(f.read())
                           f.close()
                           log.debug('TCPWave neutron listener: deleting zone level RR in the IPAM is completed successfully.')
                       except urllib.error.HTTPError as e:
                           log.debug('TCPWave neutron listener: error occured while deleting zone level RR.')
                           log.debug (e.code)
                           log.debug (e.reason)
                           log.debug (e.read())
                   else:
                       #Delete Object
                       log.debug('TCPWave neutron listener: deleting object in the IPAM.')
                       url = 'https://'+host+':7443/tims/rest/object/delete-multiple'
                       d = {   'organization_name': org,
                               'isDeleterrsChecked': 0,
                               'addressArray': [ip],
                               'subnet_id':subnet_id
                           }
                       try:
                           data = json.dumps(d)
                           req = urllib.request.Request(url=url, data=data.encode(), method='POST')
                           req.add_header('Content-Type', 'application/json')
                           f  = urllib.request.urlopen(req, context=ctx)
                           log.debug(str(f.status)+' '+f.reason)
                           log.debug(f.read())
                           f.close()
                           log.debug('TCPWave neutron listener: deleting object in the IPAM is completed successfully.')
                       except urllib.error.HTTPError as e:  
                           log.debug('TCPWave neutron listener: error occured while deleting object.')
                           log.debug (e.code)
                           log.debug (e.reason)
                           log.debug (e.read())
                
 
                         def getProjectName(project_id):
                             paramsTWC = getTWCConfig(TWC_configFileName)    
                             rest_uri = paramsTWC['twc_openstack_identity_uri']    
                             rest_url = rest_uri+"/auth/tokens?nocatalog"
                             rest_headers = {"Content-Type":"application/json"}
                             rest_data = '{ "auth": { "identity": { "methods": ["password"],"password": {"user": {"domain": {"name": "Default"},"name": "admin", "password": "tcpwave"} } }, "scope": {"project": {"domain": { "name": "Default" }, "name":  "admin" } } }}'
                          
                             response = requests.post(rest_url, data = rest_data, headers = rest_headers)
                          
                             token = response.headers["x-subject-token"]
                          
                             s_url = rest_uri+"/projects/"+project_id
                             s_headers = {"Content-Type":"application/json","x-auth-token":token}
                          
                             s_response = requests.get(s_url, headers = s_headers)
                             s_json = s_response.json()
                             project_name = s_json["project"]["name"]
                          
                        log.debug('TCPWave neutron listener: project name = %s' % project_name)
                        return project_name
                        
                        def getNetworkCIDR(network_id):
                            paramsTWC = getTWCConfig(TWC_configFileName)    
                            rest_uri = paramsTWC['twc_openstack_identity_uri']   
                            n_rest_uri = paramsTWC['twc_openstack_network_uri']
                            rest_url = rest_uri+"/auth/tokens?nocatalog"
                            rest_headers = {"Content-Type":"application/json"}
                            rest_data = '{ "auth": { "identity": { "methods": ["password"],"password": {"user": {"domain": {"name": "Default"},"name": "admin", "password": "tcpwave"} } }, "scope": {"project": {"domain": { "name": "Default" }, "name":  "admin" } } }}'
                         
                            response = requests.post(rest_url, data = rest_data, headers = rest_headers)
 
                       token = response.headers["x-subject-token"]
 
                       s_url = n_rest_uri+"/networks/"+network_id+"/tags"
                       s_headers = {"Content-Type":"application/json","x-auth-token":token}
 
                       s_response = requests.get(s_url, headers = s_headers)
                       s_json = s_response.json()
                       tags = s_json["tags"]
                       network_address = ""
                       for t in tags:
              log.debug('TCPWave neutron listener: tag = %s' % t)
              try:
                 ipaddress.ip_address(t)
                 network_address = t
              except:
                 log.debug('TCPWave neutron listener: tag '+t+' is not network address.')
 
             log.debug('TCPWave neutron listener: network address = %s' % network_address)
             return network_address
             
             ef config_parser(conf,list):
                CONF = cfg.CONF
                CONF.register_group(tcpwave_group)
                CONF.register_opts(list, "tcpwave")
                CONF(default_config_files=conf)
                return CONF
             
             lass TWCUpdater(ConsumerMixin):
             
                def __init__(self, connection):
                    self.connection = connection
                    return
             
                def get_consumers(self, consumer, channel):
                    exchange = Exchange(EXCHANGE_NAME, type='topic', durable=False)
                    queue = Queue(
                        QUEUE_NAME,
                        exchange,
                        routing_key=ROUTING_KEY,
                        durable=False,
                        auto_delete=True,
                        no_ack=True,
                        )
                    return [consumer(queue, callbacks=[self.on_message])]
             
                def on_message(self, body, message):
                    try:
                        self._handle_message(body)
                    except Exception as e:
                        log.debug(repr(e))
             
     # Message handler extracts event_type
               def _handle_message(self, body):
                   log.debug('TCPWave neutron listener: Event body = %r' % body)
                   jbody = json.loads(body['oslo.message'])
                   event_type = jbody['event_type']         
                   
                   if event_type == FLOAT_START:
                        # no relevent information in floatingip.create.start
                       log.debug ('[floatingip.create.start]')
 
                   elif event_type == FLOAT_END:
                        # only floating_ip_address in payload as IP is selected from pool
                        fixed = jbody['payload']['floatingip']['fixed_ip_address']
                        log.debug ('[floatingip.create.end] -> FIXED_IP_ADDRESS = %s' % fixed)
                        float = jbody['payload']['floatingip']['floating_ip_address']
                        log.debug ('[floatingip.create.end] -> FLOATING_IP_ADDRESS = %s' % float)
                        port_id = jbody['payload']['floatingip']['port_id']
             log.debug ('[floatingip.create.end] -> PORT_ID = %s' % port_id)
 
        elif event_type == FLOAT_U_START:
             # fixed IP from instance to which floating IP will be assigned and the port_id (upon associated)
             # NULL (upon dis-associated)
             if 'fixed_ip_address' in jbody['payload']['floatingip']:
                 fixed = jbody['payload']['floatingip']['fixed_ip_address']
                 if fixed is not None:
                     log.debug ('[floatingip.update.start] -> FIXED_IP_ADDRESS = %s' % fixed)
                     port_id = jbody['payload']['floatingip']['port_id']
                     log.debug ('[floatingip.update.start] -> PORT_ID = %s' % port_id)
 
        elif event_type == FLOAT_U_END:
             # Fixed_IP, Floating_IP and Port_ID seen (upon associate)
             # Fixed_IP = None, floating_IP, and port_id = None (upon disassociation)
             payload = jbody['payload']           
             log.debug ('TCPWave neutron listener: event =  floatingip.update.end %s' %payload)
             if 'fixed_ip_address' in jbody['payload']['floatingip']:
                fixed = jbody['payload']['floatingip']['fixed_ip_address']
                log.debug ('TCPWave neutron listener: FixedIP = %s ' % fixed)
                float = jbody['payload']['floatingip']['floating_ip_address']
                log.debug ('TCPWave neutron listener: FloatingIP = %s ' % float)
                port_id = jbody['payload']['floatingip']['port_id']
                log.debug ('TCPWave neutron listener: Port ID = %s' % port_id)
                project_id = jbody['payload']['floatingip']['project_id']
                org = getProjectName(project_id)
                log.debug ('TCPWave neutron listener: Project ID = %s' % project_id)
                name = ''
                mac = ''
                if (jbody['payload']['floatingip']['port_details'] is not None):
                    name = jbody['payload']['floatingip']['port_details']['name']
                    log.debug ('TCPWave neutron listener: Name = %s' % name) 
                    mac = jbody['payload']['floatingip']['port_details']['mac_address']
                    log.debug ('TCPWave neutron listener: Mac Address = %s' % mac)
                 
                paramsTWC = getTWCConfig(TWC_configFileName)
                zone = paramsTWC['twc_dns_zone']
                ctx = ssl.create_default_context()
                ctx.check_hostname = False
                ctx.verify_mode = ssl.CERT_NONE
                ctx.load_cert_chain(paramsTWC['twc_client_crt'], paramsTWC['twc_client_key']);
                host = paramsTWC['twc_address']
                        
                if fixed is not None and float is not None and port_id is not None:
                     allowDuplicates = False
                     url = 'https://'+host+':7443/tims/rest/globals/getGlobalOption?option_name=WARN_ON_DUPLICATE_OBJECT_NAMES'              
                     try:
                           req = urllib.request.Request(url=url, method='GET')
                           f  = urllib.request.urlopen(req, context=ctx)
                           log.debug(str(f.status))  
                           data = f.read().decode('utf-8')
                           log.debug('TCPWave neutron listener: data '+data)
                           if(data == 'No'):
                             allowDuplicates = True
                     except urllib.error.HTTPError as e:
                            log.debug('TCPWave neutron listener: error occured getting the global option value of WARN_ON_DUPLICATE_OBJECT_NAMES.')
                            log.debug (e.code)
                            log.debug (e.reason)
                            log.debug (e.read())
                     if(allowDuplicates):
                            addV4Object(float, name, mac, project_id)
                     else :
                            deleteV4Object(fixed, name, project_id)
                            addV4Object(float, name, mac, project_id)
                elif fixed is None and float is not None and port_id is None:
                  if(name == ''):
                    log.debug('TCPWave neutron listener: Getting name from the IPAM.')
                    url = 'https://'+host+':7443/tims/rest/search/search?search_type=Match&entity_type=resource_record&field_name=data&search_term='+float
                    
                    try:
                       req = urllib.request.Request(url=url, method='GET')
                       f  = urllib.request.urlopen(req, context=ctx)
                       log.debug(str(f.status)+' '+f.reason)  
                       data = json.load(f)
                       for item in data['rows']:
                           org_name = item['organization_name']
                           if( item['data'] == float and org == org_name and 'A'==item['rrtype']) :
                               name = item['owner'] 
                               if zone and name.endswith('.'+zone+'.'):
                                 name = name[:-len('.'+zone+'.')]
                       
                    except urllib.error.HTTPError as e:
                        log.debug('TCPWave neutron listener: error occured while getting name.')
                        log.debug (e.code)
                        log.debug (e.reason)
                        log.debug (e.read())
                  deleteV4Object(float, name, project_id)
 
        elif event_type == PORT_START:
             if 'id' in jbody['payload']['port']:
                 port_id = jbody['payload']['port']['id']
                 log.debug ('[port.create.start] -> PORT_ID = %s' % port_id)
 
        elif event_type == PORT_END:
             port_id = jbody['payload']['port']['id']
             log.debug ('[port.create.end] -> PORT_ID = %s' % port_id)
 
        elif event_type == PORT_U_START:
             if 'id' in jbody['payload']['port']:
                 port_id = jbody['payload']['port']['id']
                 log.debug ('[port.update.start] - > PORT_ID = %s' % port_id)
                 
        elif event_type == PORT_DELETE_END:
            payload = jbody['payload']           
            log.debug ('TCPWave neutron listener: event = port.delete.end %s' %payload)
            port = payload['port']
            project_id = port["project_id"]
            ip = port['fixed_ips'][0]['ip_address']
 
            name = ''
            name = str(port['name'])             
            if (name == ''):
               name = port['dns_assignment'][0]['hostname']
            deleteV4Object(ip, name, project_id)            
 
        elif event_type == PORT_U_END:
            paramsTWC = getTWCConfig(TWC_configFileName)
            zone = paramsTWC['twc_dns_zone']
            host = paramsTWC['twc_address']
            payload = jbody['payload']           
            log.debug ('TCPWave neutron listener: event =  port.update.end %s' %payload)
            port = payload['port']
            project_id = port["project_id"]
            ip = port['fixed_ips'][0]['ip_address']
            org = getProjectName(project_id)    
            ctx = ssl.create_default_context()
            ctx.check_hostname = False
            ctx.verify_mode = ssl.CERT_NONE
            ctx.load_cert_chain(paramsTWC['twc_client_crt'], paramsTWC['twc_client_key']);            
 
            name = ''
            device = ''
            name = str(port['name'])
            device = str(port['device_id'])
            mac = str(port['mac_address'])
            
            if (name == ''):
               name = port['dns_assignment'][0]['hostname']
 
            if (device == ''):
                ip1 = ''
                log.debug('TCPWave neutron listener: Getting IP from the IPAM.')
                url = 'https://'+host+':7443/tims/rest/search/search?search_type=Match&entity_type=resource_record&field_name=owner&search_term='+name
                
                try:
                   req = urllib.request.Request(url=url, method='GET')
                   f  = urllib.request.urlopen(req, context=ctx)
                   log.debug(str(f.status)+' '+f.reason)  
                   data = json.load(f)
                   for item in data['rows']:
                       org_name = item['organization_name']
                       if( item['owner'] == name+"."+zone+"." and org == org_name and 'A'==item['rrtype']) :
                           ip1 = item['data']
                           deleteV4Object(ip1, name, project_id)                           
                   
                except urllib.error.HTTPError as e:
                    log.debug('TCPWave neutron listener: error occured while getting name.')
                    log.debug (e.code)
                    log.debug (e.reason)
                    log.debug (e.read())
                    
                if ip1 == '':
                    url = 'https://'+host+':7443/tims/rest/search/search?search_type=Match&entity_type=object&field_name=name&search_term='+name
                
                    try:
                       req = urllib.request.Request(url=url, method='GET')
                       f  = urllib.request.urlopen(req, context=ctx)
                       log.debug(str(f.status)+' '+f.reason)  
                       data = json.load(f)
                       for item in data['rows']:
                           org_name = item['organization_name']
                           if( item['name'] == name+"." and org == org_name ) :
                               ip1 = item['address']
                               deleteV4Object(ip1, name, project_id)                           
                       
                    except urllib.error.HTTPError as e:
                        log.debug('TCPWave neutron listener: error occured while getting name.')
                        log.debug (e.code)
                        log.debug (e.reason)
                        log.debug (e.read())
                deleteV4Object(ip, name, project_id)
            else:
                addV4Object(ip, name, mac, project_id)
 
 
        elif event_type == SUBNET_CREATE_START:
            log.debug ('TCPWave neutron listener: event = subnet.create.start')
            subnet_name = jbody['payload']['subnet']['name']
            network_id = jbody['payload']['subnet']['network_id']
            tenant_id = jbody['payload']['subnet']['tenant_id']
            
            log.debug ('\033[0;32m[subnet.create.start] Subnet Name [%s] \033[1;m' % subnet_name)
            log.debug ('\033[0;32m[subnet.create.start] Network ID: %s \033[1;m' % network_id)
            log.debug ('\033[0;32m[subnet.create.start] Tenant ID: %s \033[1;m' % tenant_id)
 
            
        elif event_type == SUBNET_CREATE_END:
            payload = jbody['payload']  
            log.debug ('TCPWave neutron listener: event = subnet.create.end %s' %payload)
            subnet_name = jbody['payload']['subnet']['name']
            subnet_address = jbody['payload']['subnet']['cidr']	
            network_id = jbody['payload']['subnet']['network_id']
            tenant_id = jbody['payload']['subnet']['tenant_id']
            gateway_ip = jbody['payload']['subnet']['gateway_ip']
            subnet_id = jbody['payload']['subnet']['id']
            
            subnet_pool_id = jbody['payload']['subnet']['subnetpool_id']
            project_id = jbody['payload']['subnet']['project_id']
            
            log.debug ('TCPWave neutron listener: Subnet Name: %s ' % subnet_name)
            log.debug ('TCPWave neutron listener: Subnet Address: %s ' % subnet_address)
            log.debug ('TCPWave neutron listener: Gateway IP: %s ' % gateway_ip)
            log.debug ('TCPWave neutron listener: Network ID: %s ' % network_id)
            log.debug ('TCPWave neutron listener: Tenant ID: %s ' % tenant_id)
            log.debug ('TCPWave neutron listener: Subnet ID: %s ' % subnet_id)
            log.debug ('TCPWave neutron listener: Subnet Pool ID: %s ' % subnet_pool_id)
            log.debug ('TCPWave neutron listener: Project ID: %s ' % project_id)
            
            
            #subnet create in IPAM
            paramsTWC = getTWCConfig(TWC_configFileName)
            ctx = ssl.create_default_context()
            ctx.check_hostname = False
            ctx.verify_mode = ssl.CERT_NONE
            ctx.load_cert_chain(paramsTWC['twc_client_crt'], paramsTWC['twc_client_key']);
            host = paramsTWC['twc_address']
            org = getProjectName(project_id)
            network_address = getNetworkCIDR(network_id)
            if (network_address ==""):
              network_address = paramsTWC['twc_network_address']
            zone = paramsTWC['twc_dns_zone']            
            s = subnet_address.split('/')
            ip = s[0]
            mask_len = s[1]
            ip_bits = str(ip).split('.')
            log.debug('Add subnet')
            url = 'https://'+host+':7443/tims/rest/subnet/add'
            log.debug('TCPWave neutron listener: adding subnet in the network '+network_address+' present in the IPAM.')
            d = {       'addr1': str(ip_bits[0]),
                        'addr2': str(ip_bits[1]),
                        'addr3': str(ip_bits[2]),
                        'addr4': str(ip_bits[3]),
                        'mask_length':mask_len,
                        'name': subnet_name,
                        'routerAddress':gateway_ip,
                        'organization_name':org,
                        'network_address':network_address,
                        'primary_domain':zone,
                        'description':'Added by Openstack neutron listener.'
                }
            try:
                data = json.dumps(d)
                req = urllib.request.Request(url=url, data=data.encode(), method='POST')
                req.add_header('Content-Type', 'application/json')
                f  = urllib.request.urlopen(req, context=ctx)
                log.debug(f.status, f.reason)
                log.debug(f.read())
                f.close()
                log.debug('TCPWave neutron listener: adding subnet in the network '+network_address+' present in the IPAM is completed successfully.')
            except urllib.error.HTTPError as e:
                log.debug('TCPWave neutron listener: error occured while adding subnet in the network '+network_address+'.')
                log.debug (e.code)
                log.debug (e.reason)
                log.debug (e.read())
                
                
        elif event_type == SUBNET_DELETE_END:
            log.debug ('TCPWave neutron listener: event = subnet.delete.end')
            subnet_name = jbody['payload']['subnet']['name']
            subnet_address = jbody['payload']['subnet']['cidr']
            network_id = jbody['payload']['subnet']['network_id']
            tenant_id = jbody['payload']['subnet']['tenant_id']
            gateway_ip = jbody['payload']['subnet']['gateway_ip']
            subnet_pool_id = jbody['payload']['subnet']['subnetpool_id']
            project_id = jbody['payload']['subnet']['project_id']
            subnet_id = jbody['payload']['subnet']['id']
 
            log.debug ('TCPWave neutron listener: Subnet Name: %s ' % subnet_name)
            log.debug ('TCPWave neutron listener: Subnet Address: %s ' % subnet_address)
            log.debug ('TCPWave neutron listener: Gateway IP: %s ' % gateway_ip)
            log.debug ('TCPWave neutron listener: Network ID: %s ' % network_id)
            log.debug ('TCPWave neutron listener: Tenant ID: %s ' % tenant_id)
            log.debug ('TCPWave neutron listener: Subnet ID: %s ' % subnet_id)
            log.debug ('TCPWave neutron listener: Subnet Pool ID: %s ' % subnet_pool_id)
            log.debug ('TCPWave neutron listener: Project ID: %s ' % project_id)
            
            paramsTWC = getTWCConfig(TWC_configFileName)
            ctx = ssl.create_default_context()
            ctx.check_hostname = False
            ctx.verify_mode = ssl.CERT_NONE
            ctx.load_cert_chain(paramsTWC['twc_client_crt'], paramsTWC['twc_client_key']);
            host = paramsTWC['twc_address']
            org = getProjectName(project_id)
            log.debug('TCPWave neutron listener: deleting subnet in the IPAM.')
            #Delete subnet
            log.debug('Delete subnet')
            url = 'https://'+host+':7443/tims/rest/subnet/delete'
            d = {   'organizationName': org,
                    'isDeleterrsChecked': 0,
                    'addressList': [subnet_address]
                }
            try:
                data = json.dumps(d)
                req = urllib.request.Request(url=url, data=data.encode(), method='POST')
                req.add_header('Content-Type', 'application/json')
                f  = urllib.request.urlopen(req, context=ctx)
                log.debug(str(f.status)+' '+f.reason)
                log.debug(f.read())
                f.close()
                log.debug('TCPWave neutron listener: deleting subnet in the IPAM is completed successfully.')
            except urllib.error.HTTPError as e:
                log.debug('TCPWave neutron listener: error  occured while deleting subnet in the IPAM.')
                log.debug (e.code)
                log.debug (e.reason)
                log.debug (e.read())
                
        elif event_type == SUBNET_UPDATE_END:
            log.debug ('\033[0;32m[subnet.update.end]\033[1;m')
        elif event_type == NETWORK_CREATE_START:
            log.debug ('\033[0;32m[network.create.start]\033[1;m')
        elif event_type == NETWORK_CREATE_END:
            log.debug ('\033[0;32m[network.create.end]\033[1;m')
        elif event_type == NETWORK_DELETE_END:
            log.debug ('\033[0;32m[network.delete.end]\033[1;m')
        elif event_type == NETWORK_UPDATE_END:
            log.debug ('\033[0;32m[network.update.end]\033[1;m')
 
        if __name__ == "__main__":
               log.debug("TCPWave neutron listener: version %s TCPWave Networks 2022" % version)
               log.debug("TCPWave neutron listener: AMQ connection URI: %s" % monitor_broker)
            
               with BrokerConnection(monitor_broker) as connection:
                   try:
                       log.debug(connection)
                       TWCUpdater(connection).run()
                   except KeyboardInterrupt:
                       log.debug('TCPWave neutron listener: exiting TCPWave Neutron Monitor..')
     
  Create file **/usr/lib/systemd/system/neutron-listener.service** with below content
     
          [Unit]
          Description=TCPWave neutron event handler
          After=neutron-server.service
           
          [Service]
          Type=simple
          User=root
          ExecStart=/usr/bin/python3 /opt/tcpwave/neutron_monitor.py
          Restart=on-failure
           
          [Install]
          WantedBy=multi-user.target
          
  Execute the commands: **systemctl daemon-reload** and **systemctl start neutron-listener**
  
  After following the above steps, neutron-listener service will be up and running.
  
  The amq messages from neutron service will be monitored.
  
  The neutron-listener logs can be seen using the command:  **tail -f /var/log/neutron/server.log | grep 'TCPWave neutron listener’**
  
# Use cases

   1. Create/delete a subnet in Openstack, then static subnet will be added/deleted from the IPAM.
   2.  Create an instance in Openstack with network address, then an object will be created in the IPAM.             
   3. Attach interface to the Openstack instance, then an object will be created in the IPAM.
   4. Associate a floating IP to the Openstack instance, then an object will be created in the IPAM.
   5. Disassociate floating IP from the Openstack instance, then object will be deleted from the IPAM.
   6. Detach interface from the Openstack instance, then an object will be deleted from the IPAM.
   7. Delete the Openstack instance, then an object will be deleted from the IPAM.
   
In all the above cases, object will be created in the IPAM when the subnet is present in the IPAM. Otherwise, a resource record will be created in the zone specified in the /etc/neutron/neutron.conf file.  When a floating IP is associated with an instance, object will be added in the IPAM with e same name as the interface to which floating IP is associated when the global option ‘Warn on duplicate object names’ is set to No. Otherwise, object will be overridden.

# Conclusion

By following the above configuration, OpenStack automation will be achieved to manage DNS entries without manual intervention when servers are allocated or deallocated in OpenStack using the neutron-listener service implemented by TCPWave.

          
 

           
  
  



 
