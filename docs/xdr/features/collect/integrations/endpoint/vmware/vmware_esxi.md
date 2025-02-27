uuid: 2b13307b-7439-4973-900a-2b58303cac90
name: VMWare ESXi
type: intake

## Overview

VMware ESXi is a hypervisor and an operation system. It serves virtual computers while running directly on the server hardware

!!! warning
    This format is still in beta, please use it wisely.


{!_shared_content/operations_center/detection/generated/suggested_rules_2b13307b-7439-4973-900a-2b58303cac90_do_not_edit_manually.md!}

{!_shared_content/operations_center/integrations/generated/2b13307b-7439-4973-900a-2b58303cac90.md!}

## Configure

### Prerequisites

An internal log concentrator is required to collect and forward events to SEKOIA.IO.

### Enable Syslog forwarding

Browse to the host thank to the vSphere Client inventory and follow [this guide](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.esxi.upgrade.doc/GUID-9F67DB52-F469-451F-B6C8-DAE8D95976E7.html) to enable the log forwarding to the log concentrator.

## Create the intake

Go to the [intake page](https://app.sekoia.io/operations/intakes) and create a new intake from the format `VMWare ESXi`.

## Transport to SEKOIA.IO

Please consult the [Syslog Forwarding](../../../../ingestion_methods/sekoiaio_docker_concentrator/) documentation to forward these logs to SEKOIA.IO.
