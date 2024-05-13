---
dg-publish: true
dg-home: 
tags:
  - software
  - parallel-processing
  - data-processing
---
> [!abstract] Map
> This component is responsible primarily for creating the visualisations. However, to do this, it has several sub functions
> 1. Consuming information from socket io
> 2. Room determination based on coordinate
> 3. Logic system to update colour back to original colour
> 4. Visualisation logic

## Consuming information from Socket io

Define the socket in `components/Context/socket.js`

```javascript
import { io } from 'socket.io-client';
export const socket = io('http://localhost:8080');
```

In the `MyMap` component, if socket is connected, read from all sockets and parse data based on the flags.

```javascript
useEffect(() => {
	if(socket){
		socket.onAny((flag, coodinput) => {
```

## Room determination based on coordinates

If a new tag data packet arrives, the room/[GeoJSON](https://geojson.org) layer the coordinate is currently in needs to be determined. 

Parse the JSON data previously produced by [[Server/Apache Flink\|Apache Flink]].

```javascript
let cood: FlinkData = JSON.parse(coodinput);
```

Check each layer current rendered on the [Leaflet](https://leafletjs.com) Map with the [GeoJSON](https://geojson.org) file to see if the coordinate is a point in the polygon. Here `PointInPolygon` is a standard math equation checking for boundary crossing conditions to determine if the current point belongs to a polygon.

```javascript
mapref.eachLayer((layer) => {
	let room = roomPoly.get(layer)
	if (!room){return;}
	
	if (PointInPolygon(room.cood, cood)) {
        let ogRoom = tagMap.get(cood.tagid);
```

If the point belongs to the polygon, the point is then checked for specifics - the detachment status and it's old room status. Colours and tags of layer objects are then updated accordingly.

## Logic system to reset room colour

Apart from colouring it when an abnormality is spotted, the layers also need to reset when the coloured box is no longer need. A counter is implemented. 

The object `roomBool` saves the `roomName` and the colour count of the room. If the count hits the max value (3) , the room will be updated back to it's original colour. Otherwise, the count will increase with each message passed or left untouched.

```javascript
const [roomBool, setRoomBool] = useState(new Map<string,number>()); // used to set data back
let count = roomBool.get(roomName)
	switch(count){
		case undefined:
			roomBool.set(roomName, 0)
		case 3:
			roomBool.set(roomName, 0)
			nlayer = UpdateColour(layer, room, "yellow")
		case 0:
			console.log("in room count 0", count)
		case 1:
			console.log("in room count 1", count)
			if (count !== undenined){
				roomBool.set(roomName, count+1)
			}
		case 2:
			console.log("in room count 2", count)
			if (count !== undefined){
				roomBool.set(roomName, count+1)
			}
	setRoomBool(roomBool)
```

## Visualisation logic

The update colour function is as follows. The function takes in the layer it is to manipulate, the room information and the colour it needs to update. 

The original layer is removed:

```javascript
mapref.removeLayer(layer);
```

The polygon is then rebuilt with [Leaflet](https://leafletjs.com) `LatLngExpression` objects and added to the current rendered map, creating the update colour effect.

```javascript
function UpdateColour(layer:L.Layer, room: Room, color:string){

	mapref.removeLayer(layer);
	let rebuiltPolygon:L.LatLngExpression[] = MakeLatLng(room.cood)
	let nlayer = L.polygon(
		rebuiltPolygon, {
		fillColor: color,
		fillOpacity:1,
		color: "black",
		weight: 2})

	mapref.addLayer(nlayer);
	return nlayer;

}
```