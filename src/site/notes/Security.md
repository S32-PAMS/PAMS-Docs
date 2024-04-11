---
dg-publish: true
dg-home: 
tags:
  - security
---
> [!abstract] Security design and considerations
> This document outlines the security designs put into the system.

Implementation manual: [[Guides/For security personnel\|For security personnel]]

# Safety considerations

## Code safety

> [!tldr]
> Code safety practices should be enforced.

Coding standards and conventions:
- Ensure code readability and maintainability
- Adopt coding standards

Code reviews:
- To adhere to coding standards and share knowledge across the team
- Mandatory peer reviews before pushing and merging code changes using GitHub for collaboration

Secure coding practices
- No hardcoded credentials (*WIP*)
- Validate and sanitise input data to prevent injection attacks
- Implement proper authentication (OAuth) and authorisation checks
- Use secure communication protocols (TLS)
- Keep dependencies up-to-date (for client to continue doing after handover of project)

## Environment safety

> [!tldr]
> These are considerations put in place to ensure environmental safety in software development for our server side.

Docker authentication:
- Controls who can push and pull images, to ensure only authorised images are used in our environments

Image security
- Minimal images that contain only necessary packages and dependencies to reduce attack surface

(To consider for remote deployment) AWS Authentication


## Link safety

> [!tldr]
> These are the considerations put in place to ensure link safety. It is possible to implement PKI and TLS to further secure the system as well.b

- Ultra-Wideband
	- Time of Flight and Ranging, while allowing functional calculation of distance between tag and anchor, also verifies the physical proximity of devices, making it challenging for attackers to spoof a device's location
- WiFi
	- This depends on network environment, but if client uses secure WiFi which already has encryption (at this moment preferably WPA3), you can be assured that the link between anchors and MQTT broker is secured to link layer level
- WebSocket
	- Inherent security with Same-Origin Policy (SOP) to restrict how a document or script loaded from one origin can interact with resources from another origin, reducing chances of Cross-Site Scripting (XSS) attacks
	- Has support for Cookies and HTTP authentication headers if it is decided to be implemented in the future


## Hardware safety

> [!tldr]
> These are the design considerations to ensure hardware safety, which includes tamper-proofing and the option to pass messages with TLS secured links.

- [[Tags\|Tags]]
	- Physical security
		- Accelerometer and force sensor prevents physical tampering by giving signals on movement and detachment
	- UWB link
		- Time of Flight and Ranging, while allowing functional calculation of distance between tag and anchor, also verifies the physical proximity of devices, making it challenging for attackers to spoof a device's location
- [[Anchors\|Anchors]]
	- Physical security
		- Secure installation (should install anchors in locations not easily accessible, up to the user)
	- WiFi + MQTT link
		- Have option to use MQTTS or MQTT, which has added TLS encryption for link encryption (WIP)

## Backend safety

> [!tldr]
> These are the considerations you should take note of on the backend.

- Database
	- Data encryption
	- Database administrator authentication
	- Parametrised queries/Prepared statements (Prevent SQL Injection) (wip)
- Web server
	- TLS (wip)

## Frontend safety

> [!tldr]
> These are the considerations you should take note of on the frontend.

- User authentication OAuth JWT (login system)
- Input sanitisation and validation (wip)
- Role-Based Access Control (admin superuser, normal user) : principle PoLP (wip)

