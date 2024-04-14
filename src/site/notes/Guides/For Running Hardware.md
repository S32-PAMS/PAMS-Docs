---
{"dg-publish":true,"permalink":"/guides/for-running-hardware/","tags":["hardware"],"noteIcon":""}
---

> [!abstract] How to set up hardware for prototype
> The anchors and tags are the hardware main components that act as the base unit of the system as such they play an important role in our system. The following are instructions on how to use the anchors and tags once they have been setup.

This assume you have built and setup all hardware. If not, refer to [[Guides/For Building PAMS#Hardware\|For Building PAMS#Hardware]].
## Anchor and Tag Initialisation

The anchors need to be plugged in via the **USB C to a power source of 5A max**. This can be done by plugging into a wall socket with an appropriate connector.

The tags need to be **charged** before they can be used. This can be done using the **USB C** port of the Battery Charging module. The Tag will require approximately *3 to 4 hours* before it becomes fully charged. The Battery charging module *flashes a blue LED* when the battery is fully charged.

## Anchor Placement

The Anchors need to be placed in ideal locations throughout the space that needs to be monitored. This is to ensure that blind spots are not created in the room and the tag can be always seen at any given point.

For the purposes of understanding how placement of the tags would work we have illustrated a few examples of how Anchor placement can be done.

### 4x4 Room

In a standard square 4x4 room, the diagonal of the room has a length of 5.65m. Since this is less than our maximal radius of 7m required for anchor measurement, placing an anchor in the 3 corners respectively would provide sufficient information to determine if a tag is in a room (Refer to Figure 1).

![Room Placement]( Room_Placement.jpg "Room Placement")

*Figure 1: Sample placement of Tags in a 4x4 room where the three anchors are signified by the boxes marked 1, 2 and 3*

### Corridor

In a larger system such as the 20m corridor shown, due to the high coverage of our 7m radii anchors, it can be sparsely placed. 

> [!note]
> The anchors within the room also contribute to the tracking of the tags in the corridor, but between the walls there is still an interference of about 2.5m. However since this does not affect the localisation of the room where the Tag is placed, it is inconsequential.

![Corridor Placement]( Corridor_Placement.jpg "Corridor Placement")

*Figure 2: Sample Placement of Tags in a corridor 20m long where the anchors are signified by the boxes numbered 1 to 6*

As we can see the penetration of the UWB signal through the thinner walls can allow for localisation even in L-shaped spaces between corridors.

Once the anchors are placed in their designated locations which allow them to be connected to the necessary power source, the anchor coordinates should be input from the frontend to allow for effective and accurate localisation. These coordinates must be put in according to their relative coordinate system.

## Tag Attachment

There are two types of Tags available

- With Detachment Latch
- Without Detachment Latch

### Tags with Detachment Latch

Once the tags are fully charged the tags with Detachment latch can simply be latched on to the object of interest. In the interest of safety choose a discrete location to attach the latch if possible to prevent detection of the tag by potential thieves.

### Tags without Detachment Latch

Tags without Detachment latches can simply be inserted into the object of interest (if such a hollow space is available) or placed on the outside of the object using some adhesive material.

> Once the tags and anchors are placed in their suitable location and the software is up and running, PAMS is now working to keep your object safe from theft 24/7.

- Now proceed to set up server prototype if you haven't: [[Guides/For Running Server\|For Running Server]]
- Back to [[README\|README]]