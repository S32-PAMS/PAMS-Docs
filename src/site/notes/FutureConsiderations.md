---
{"dg-publish":true,"permalink":"/FutureConsiderations/","tags":["FutureConsiderations","Hardware","Software(Frontend)"],"noteIcon":""}
---


Considerations for future modifications for PAMS Tags and Anchors

# Hardware
## General improvements

### Custom PCBs
One of the ways to improve both the tags and the anchors is to create custom PCBs with all surface-mounted components instead of soldered components. This effectively reduces the footprint of the devices hence reducing its size.

## Tag improvements
### Improved Antenna
An issue that we have faced is the inconsistencies due to signal transmissions in noisy environments. As signal transmissions drastically improve during line-of-sight, an improvement in the antennas,  would reduce line-of-sight inconsistencies, boosting capabilities of the tag. 

This can be done by installing a separate antenna possibly on the external casing of the tag. Do note that because of the frequency difference between the 2 wireless protocols, there might be a need for 2 different antennas.
### Mounting
An issue that needs to be addressed is the method of mounting the tags onto the item of interest. With the miniaturization of the tag, this issue would be significantly easier to solve. 
### Detachment Mechanism
As of now, we are using a physical detachment mechanism with a piezoelectric pressure sensor to detect detachment of the tag from the item of interest. 

To improve this, we recommend these 2 possibilities.
1. Changing the mechanism to vertical-cavity surface-emitting lasers (VCSEL) as a proximity sensor.
2. Changing the mechanism to VCSEL as a capacitive sensor. This solution may be affected by many constants.

## Anchor Improvements
### Improved Antenna
An issue that we have faced is the inconsistencies due to signal transmissions in noisy environments. As they drastically improve during line-of-sight, it shows that with an improvement in the antennas, it would boost the capability of the tags by reducing such inconsistencies. 

This can be resolved by installing a separate antenna. As the anchors are mounted in the environment itself, standard omnidirectional antennas should be enough. Do note that because of the frequency difference between the 2 wireless protocols, there might be a need for 2 different antennas.

# Software (Frontend)
## Visualisation
Previously we have established the camera as a means to prevent hallucinations. However, the addition of the camera into the system implies further possibilities. Below we shall illustrate these possibilities.
### Mass Camera Viewing
A detection of movement outside of the boundary allocated will trigger a message with the room the tag is currently in as well as a list of cameras in said room. 
![cameraAlertsOnMap.png](/img/user/Attachments/frontend-users/cameraAlertsOnMap.png)

Users can navigate to the a page with the list of cameras and find relevant cameras to understand the situation in these rooms. 
![cameraListPage.png](/img/user/Attachments/frontend-users/cameraListPage.png)