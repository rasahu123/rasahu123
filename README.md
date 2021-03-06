- š Hi, Iām @rasahu123
- š Iām interested in ...
- š± Iām currently learning ...
- šļø Iām looking to collaborate on ...
- š« How to reach me ...

<!---
rasahu123/rasahu123 is a āØ special āØ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
import requests
from requests import Session
from requests.auth import HTTPBasicAuth
from zeep import Client, Settings
from zeep.transports import Transport
from zeep.plugins import HistoryPlugin
from zeep.exceptions import Fault
from xml.etree import ElementTree as ET
from texttable import Texttable as TT
import os

# Function to validate Cisco server application is accessable
def server_check(server):
    try:
        reply = requests.get("https://{}:8443/".format(server), verify=False, timeout=3)
    # Test if server application is accessable within 3 seconds
    except requests.exceptions.RequestException:
        return False
    # If server check receives valid "200" response, break out of iteration
    else:
        if reply.status_code == 200:
            return True
        else:
            return False

# Function to set up the AXL call to CUCM
def axl_bind(server, acct, pw):
    binding_name = "{http://www.cisco.com/AXLAPIService/}AXLAPIBinding"
    axl_address = "https://{}:8443/axl/".format(server)
    script_dir = os.path.dirname(__file__)
    wsdl_file = "{}\\AXLAPI.wsdl".format(script_dir)
    session = Session()
    session.verify = False
    session.auth = HTTPBasicAuth(acct, pw)
    transport = Transport(session=session, timeout=20)
    history = HistoryPlugin()
    settings = Settings(strict=False, xml_huge_tree=True)
    client = Client(wsdl=wsdl_file, transport=transport, plugins=[history], settings=settings)
    axl = client.create_service(binding_name, axl_address)


    return axl, history

# CUCM updateUser call to change pin
def pin_set(server, acct, pw, user, pin):
    axl, history = axl_bind(server, acct, pw)
    # Attempt AXL call
    try:
        reply = axl.updateUser(userid=user, pin=pin)
    # Check for error
    except Fault as err:
        if 'HTTP Status 401' in ET.tostring(history.last_received['envelope']).decode():
            return "Server Credential Error"
        else:
            return err
    # Return values based on request response
    else:
        if reply["return"] is None:
            return "Unknown Result - Double Check Values"
        else:
            return "Pin Successfully Reset"

# CUCM updateServiceParameter call to change FNA timer
def fna_set(server, acct, pw, time):
    axl, history = axl_bind(server, acct, pw)
    # Attempt AXL call
    try:
        reply = axl.updateServiceParameter(processNodeName="EnterpriseWideData", service="Cisco CallManager", name="ForwardNoAnswerTimeout", value=time)
    # Check for error
    except Fault as err:
        if 'HTTP Status 401' in ET.tostring(history.last_received['envelope']).decode():
            return "Server Credential Error"
        else:
            return err
    # Return values based on request response
    else:
        if reply["return"] is None:
            return "Unknown Result - Double Check Values"
        else:
            return "Forward No Answer Timer Successfully Set"

# CUCM listRoutePlan call to print route plan report
def route_print(server, acct, pw):
    axl, history = axl_bind(server, acct, pw)
    # Attempt AXL call
    try:
        reply = axl.listRoutePlan(searchCriteria={"dnOrPattern": "%"}, returnedTags={"dnOrPattern": "", "partition": "", "type": "", "routeDetail": ""})
    # Check for error
    except Fault as err:
        if 'HTTP Status 401' in ET.tostring(history.last_received['envelope']).decode():
            return "Server Credential Error"
        else:
            return err
    # Return values based on request response
    else:
        if reply["return"] is None:
            return "Unknown Result - Double Check Values"
        else:
            # Parse route plan report data into list of lists
            report = []
            for route in reply["return"]["routePlan"]:
                item = [route["dnOrPattern"], route["partition"]["_value_1"], route["type"], route["routeDetail"]]
                report.append(item)
            # Sort lists by pattern
            report.sort(key=lambda pattern: pattern[0])
            # Set report header
            report = [["Pattern", "Partition", "Type", "Route Detail"]] + report
            # Create report
            table = TT()
            # Set all columns to "text"
            table.set_cols_dtype(['t', 't', 't', 't'])
            table.add_rows(report)
            return table.draw()

# CUCM SQL query to check call forward setting
def cfa_check(server, acct, pw, dn, pt):
    axl, history = axl_bind(server, acct, pw)
    attrib = """select numplan.dnorpattern as pattern, routepartition.name as partition,
                callforwarddynamic.cfadestination as destination, callingsearchspace.name as css from numplan 
                inner join routepartition on numplan.fkroutepartition=routepartition.pkid
                inner join callforwarddynamic on numplan.pkid=callforwarddynamic.fknumplan 
                left join callingsearchspace on callforwarddynamic.fkcallingsearchspace_cfa=callingsearchspace.pkid
                where numplan.dnorpattern='{}' and routepartition.name='{}'""".format(dn, pt)
    # Attempt AXL call
    try:
        reply = axl.executeSQLQuery(sql=attrib)
    # Check for error
    except Fault as err:
        if 'HTTP Status 401' in ET.tostring(history.last_received['envelope']).decode():
            return "Server Credential Error"
        else:
            return err
    # Return values based on request response
    else:
        if reply["return"] is None:
            return "Unknown Result - Double Check Values"
        else:
            report = []
            header = []
            data = []
            for result in reply["return"]["row"]:
                for element in result:
                    header.append(element.tag)
                    data.append(element.text)
            report.append(header)
            report.append(data)
            # Create report
            table = TT()
            # Set all columns to "text"
            table.set_cols_dtype(['t', 't', 't', 't'])
            table.add_rows(report)
            return table.draw()

# CUCM SQL update to change call forward setting
def cfa_set(server, acct, pw, dn, pt, dest, css):
    axl, history = axl_bind(server, acct, pw)
    attrib = """update callforwarddynamic set cfadestination='{}',
                fkcallingsearchspace_cfa=(select pkid from callingsearchspace where name='{}') 
                where fknumplan in (select pkid from numplan where dnorpattern='{}' and
                fkroutepartition in (select pkid from routepartition where name='{}'))""".format(dest, css, dn, pt)
    # Attempt AXL call
    try:
        reply = axl.executeSQLUpdate(sql=attrib)
    # Check for error
    except Fault as err:
        if 'HTTP Status 401' in ET.tostring(history.last_received['envelope']).decode():
            return "Server Credential Error"
        else:
            return err
    # Return values based on request response
    else:
        if reply['return']['rowsUpdated'] < 1:
            return "Unknown Result - Double Check Values"
        else:
            return "Call Forward All Successfully Set"

# CUCM SQL update to change device MAC address
def mac_set(server, acct, pw, old_mac, new_mac):
    axl, history = axl_bind(server, acct, pw)
    attrib = """update device set name='SEP{}' where name='SEP{}'""".format(new_mac, old_mac)
    # Attempt AXL call
    try:
        reply = axl.executeSQLUpdate(sql=attrib)
    # Check for error
    except Fault as err:
        if 'HTTP Status 401' in ET.tostring(history.last_received['envelope']).decode():
            return "Server Credential Error"
        else:
            return err
    # Return values based on request response
    else:
        if reply['return']['rowsUpdated'] < 1:
            return "No Device with MAC {}".format(old_mac)
        else:
            return "MAC Address Updated to {}".format(new_mac)
            
