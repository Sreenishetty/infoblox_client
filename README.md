# Infoblox_Client
Client for interacting with Infoblox NIOS over WAPI.
  - Free software: Apache license
  - Documentation: https://infoblox-client.readthedocs.org/
  
 # Installation
Install infoblox-client using pip:
```
pip install infoblox-client
```

# Usage
Configure logger prior to loading infoblox_client to get all debug messages in console:
```
import logging
logging.basicConfig(level=logging.DEBUG)
```

# Low level API, using connector module
Retrieve list of network views from NIOS:
```
from infoblox_client import connector

opts = {'host': '192.168.1.10', 'username': 'admin', 'password': 'admin'}
conn = connector.Connector(opts)
# get all network_views
network_views = conn.get_object('networkview')
# search network by cidr in specific network view
network = conn.get_object('network', {'network': '100.0.0.0/8', 'network_view': 'default'})
```
For these request data is returned as list of dicts:
```
network_views:
[{u'_ref': u'networkview/ZG5zLm5ldHdvcmtfdmlldyQw:default/true',
  u'is_default': True,
  u'name': u'default'}]

network:
[{u'_ref': u'network/ZG5zLm5ldHdvcmskMTAwLjAuMC4wLzgvMA:100.0.0.0/8/default',
  u'network': u'100.0.0.0/8',
  u'network_view': u'default'}]
```
## High level API, using objects
Example of creating Network View, Network, DNS View, DNSZone and HostRecord using NIOS objects:
```
from infoblox_client import connector
from infoblox_client import objects

opts = {'host': '192.168.1.10', 'username': 'admin', 'password': 'admin'}
conn = connector.Connector(opts)
```

Create a network view, and network:
```
nview = objects.NetworkView.create(conn, name='my_view')
network = objects.Network.create(conn, network_view='my_view', cidr='192.168.1.0/24')
```

Create a DNS view and zone:
```
view = objects.DNSView.create(conn, network_view='my_view', name='my_dns_view')
zone = objects.DNSZone.create(conn, view='my_dns_view', fqdn='my_zone.com')
```
Create a host record:
```
my_ip = objects.IP.create(ip='192.168.1.25', mac='aa:bb:cc:11:22:33')
hr = objects.HostRecord.create(conn, view='my_dns_view',
                               name='my_host_record.my_zone.com', ip=my_ip)
```

Create host record with Extensible Attributes (EA):
```
ea = objects.EA({'Tenant ID': tenantid, 'CMP Type': cmptype,
                 'Cloud API Owned': True})
host = objects.HostRecord.create(conn, name='new_host', ip=my_ip, extattrs=ea)
```

Set the TTL to 30 minutes:
```
hr = objects.HostRecord.create(conn, view='my_dns_view',
                               name='my_host_record.my_zone.com', ip=my_ip,
                               ttl = 1800)
```                               

Create a new host record, from the next available IP in a CIDR, with a MAC address, and DHCP enabled:
```
next = objects.IPAllocation.next_available_ip_from_cidr('default', '10.0.0.0/24')
my_ip = objects.IP.create(ip=next, mac='aa:bb:cc:11:22:33', configure_for_dhcp=True)
host = objects.HostRecord.create(conn, name='some.valid.fqdn', view='Internal', ip=my_ip)
```

Reply from NIOS is parsed back into objects and contains next data:
```
In [22]: hr
Out[22]: HostRecordV4: _ref=record:host/ZG5zLmhvc3QkLjQuY29tLm15X3pvbmUubXlfaG9zdF9yZWNvcmQ:my_host_record.my_zone.com/my_dns_view, name=my_host_record.my_zone.com, ipv4addrs=[<infoblox_client.objects.IPv4 object at 0x7f7d6b0fe9d0>], view=my_dns_view
```

Create a new fixed address, with a MS server DHCP reservation:
```
obj, created = objects.FixedAddress.create_check_exists(connector=conn,
                                                        ip='192.168.100.100',
                                                        mac='aa:bb:cc:11:22:33',
                                                        comment='My DHCP reservation',
                                                        name='My hostname',
                                                        network_view='default',
                                                        ms_server={'_struct': 'msdhcpserver',
                                                                   'ipv4addr': '192.168.0.0'})
```

# High level API, using InfobloxObjectManager
Create a new fixed address, selecting it from the next available IP in a CIDR:
```
from infoblox_client.object_manager import InfobloxObjectManager

new_address = InfobloxObjectManager(conn).create_fixed_address_from_cidr(netview='default', mac='aa:bb:cc:11:22:33', cidr='10.0.0.0/24', extattrs=[])

```
What you get back is a ```FixedAddressV4``` object.

## Objects Interface
All top level objects support interface for CRUD operations. List of supported objects is defined in next section.

- ```##create(cls, connector, check_if_exists=True, update_if_exists=False, **kwargs)##```
Creates object on NIOS side. Requires connector passed as the first argument, check_if_exists and update_if_exists are optional. Object related fields are passed in as kwargs: field=value, field2=value2.

