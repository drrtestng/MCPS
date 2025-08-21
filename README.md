<!DOCTYPE html>
<html>
<head>
  <title>Resource Monitoring and Tracking</title>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <link rel="stylesheet" href="https://unpkg.com/leaflet-control-geocoder/dist/Control.Geocoder.css" />

  <style>
    body { margin: 0; font-family: Arial, sans-serif; height: 100vh; display: flex; flex-direction: column; }
    #navbar { background: #007bff; color: white; padding: 10px 20px; font-size: 20px; font-weight: bold; }
    #top { display: flex; flex: 2; padding: 10px; gap: 10px; min-height: 50%; }
    #map { flex: 2; height: 100%; border: 1px solid #ccc; border-radius: 8px; position: relative; }
    #sidebar { flex: 1; background: #f8f8f8; border-left: 2px solid #ccc; padding: 10px; overflow-y: auto; border-radius: 8px; }
    .header { text-align: center; font-size: 20px; font-weight: bold; margin-bottom: 10px; }
    .responder { padding: 10px; margin: 5px 0; background: white; border: 1px solid #ddd; border-radius: 6px; cursor: pointer; }
    .responder:hover { background: #e6f0ff; }
    #messaging { flex: 1; background: #f0f0f0; padding: 15px; border-top: 2px solid #ccc; margin: 10px; border-radius: 8px; }
    #messaging h3 { text-align: center; margin-bottom: 10px; }
    #numbers, #message, #template, #barangay, #street, #landmark, #callback { width: 100%; padding: 10px; margin: 5px 0; border-radius: 8px; border: 1px solid #ccc; font-size: 14px; }
    .row { display: flex; gap: 10px; }
    .row select, .row input { flex: 1; }
    #sendBtn { display: block; margin: 10px auto; padding: 10px 20px; background: #007bff; color: white; border: none; border-radius: 8px; cursor: pointer; }
    #sendBtn:hover { background: #0056b3; }
    #stats { background: white; border-top: 2px solid #ccc; padding: 15px; margin: 10px; border-radius: 8px; }
    #stats h3 { text-align: center; margin-bottom: 10px; }
    .stat { padding: 10px; margin: 5px 0; background: #f9f9f9; border: 1px solid #ddd; border-radius: 6px; }
    #barangayCount { display: flex; flex-direction: column; gap: 5px; }
    .leaflet-control-geocoder { z-index: 1000; }
    .leaflet-control-geocoder input { width: 200px; }
  </style>
</head>
<body>
  <div id="navbar">Resource Monitoring & Tracking System</div>

  <div id="top">
    <div id="map"></div>
    <div id="sidebar">
      <div class="header">NEAREST RESPONDER</div>
      <div id="responderList"></div>
    </div>
  </div>

  <div id="messaging">
    <h3>MESSAGING AND ALERTING</h3>
    <input type="text" id="numbers" readonly placeholder="Selected responder numbers">
    <div class="row">
      <select id="template">
        <option value="">-- Select Emergency Type --</option>
        <option value="POLICE EMERGENCY">🚔 Police Emergency</option>
        <option value="FIRE EMERGENCY">🔥 Fire Emergency</option>
        <option value="MEDICAL EMERGENCY">🚑 Medical Emergency</option>
        <option value="TRAFFIC ACCIDENT">🚧 Traffic Accident</option>
        <option value="RESCUE OPERATION">🛟 Rescue Operation</option>
      </select>
    </div>
    <div class="row">
      <select id="barangay">
        <option value="">-- Select Barangay --</option>
        <option value="Alabang">Alabang</option>
        <option value="Ayala Alabang">Ayala Alabang</option>
        <option value="Bayanan">Bayanan</option>
        <option value="Buli">Buli</option>
        <option value="Cupang">Cupang</option>
        <option value="Poblacion">Poblacion</option>
        <option value="Putatan">Putatan</option>
        <option value="Sucat">Sucat</option>
        <option value="Tunasan">Tunasan</option>
      </select>
      <input type="text" id="street" placeholder="Street Name">
    </div>
    <div class="row">
      <input type="text" id="landmark" placeholder="Nearest Landmark">
      <input type="text" id="callback" placeholder="Caller Callback Number">
    </div>
    <textarea id="message" rows="4" placeholder="Type emergency message here..."></textarea>
    <button id="sendBtn">Send Alert</button>
  </div>

  <div id="stats">
    <h3>📊 Incident Statistics</h3>
    <div class="stat">🚔 Police Emergencies: <span id="policeCount">0</span></div>
    <div class="stat">🏘️ Incidents Per Barangay: <span id="barangayCount"></span></div>
    <div class="stat">✅ Total Responded: <span id="totalResponded">0</span></div>
  </div>

  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script src="https://unpkg.com/leaflet-control-geocoder/dist/Control.Geocoder.js"></script>
  <script>
    var map = L.map('map').setView([14.4081, 121.0415], 14);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19 }).addTo(map);
    var muntinlupaBounds = L.latLngBounds([14.3610, 120.9750],[14.4750, 121.0920]);

    var geocoderControl = L.Control.geocoder({
      defaultMarkGeocode: false,
      placeholder: "Search location in Muntinlupa...",
      geocoder: new L.Control.Geocoder.Nominatim()
    }).addTo(map);

    var searchMarker;

    function handleGeocodeResult(geocode) {
      var center = geocode.center;
      if (!muntinlupaBounds.contains(center)) {
        alert("⚠️ Please select a location inside Muntinlupa only.");
        return;
      }
      if (searchMarker) map.removeLayer(searchMarker);

      var redIcon = new L.Icon({
        iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-red.png',
        shadowUrl: 'https://unpkg.com/leaflet@1.9.3/dist/images/marker-shadow.png',
        iconSize: [25, 41],
        iconAnchor: [12, 41],
        popupAnchor: [1, -34],
        shadowSize: [41, 41]
      });

      searchMarker = L.marker(center, {icon: redIcon})
                      .addTo(map)
                      .bindPopup("📍 " + geocode.name)
                      .openPopup();

      map.setView(center, 15);
      updateNearestResponders(center.lat, center.lng);
    }

    geocoderControl.on('markgeocode', function(e) {
      handleGeocodeResult(e.geocode);
    });

    // Wait until DOM is ready
    setTimeout(function(){
      var geocoderInput = document.querySelector('.leaflet-control-geocoder-form input');
      var geocoderButton = document.querySelector('.leaflet-control-geocoder-form a');

      // Enter key
      geocoderInput.addEventListener('keydown', function(e){
        if(e.key === 'Enter'){
          e.preventDefault();
          geocoderControl.options.geocoder.geocode(this.value, function(results){
            if(results.length > 0) handleGeocodeResult(results[0]);
            else alert("No results found!");
          });
        }
      });

      // Search button click
      geocoderButton.addEventListener('click', function(e){
        e.preventDefault();
        var query = geocoderInput.value.trim();
        if(query){
          geocoderControl.options.geocoder.geocode(query, function(results){
            if(results.length > 0) handleGeocodeResult(results[0]);
            else alert("No results found!");
          });
        }
      });
    }, 500);

    // Responders
    var responders = [
      { id: 1, name: "Juan Dela Cruz", mobile: "0999999999", coords: [14.4081, 121.0415] },
      { id: 2, name: "Juan Carlo Dela Cruz", mobile: "0999999989", coords: [14.4120, 121.0450] },
      { id: 3, name: "Juan Manuel Cruz", mobile: "0999999979", coords: [14.4040, 121.0500] },
      { id: 4, name: "Juan Way Cruz", mobile: "0999999969", coords: [14.4150, 121.0380] }
    ];

    var markers = {}, selectedNumbers = [];
    responders.forEach(function(r){
      var marker = L.marker(r.coords).addTo(map).bindPopup(`<b>${r.name}</b><br>${r.mobile}`);
      markers[r.id] = marker;

      var div = document.createElement("div");
      div.className = "responder";
      div.innerHTML = `<b>GeoTag #${r.id}</b><br>Name: ${r.name}<br>Mobile: ${r.mobile}`;
      div.onclick = function() {
        map.setView(r.coords, 16); marker.openPopup();
        if (!selectedNumbers.includes(r.mobile)) {
          selectedNumbers.push(r.mobile);
          document.getElementById("numbers").value = selectedNumbers.join(", ");
        }
      };
      document.getElementById("responderList").appendChild(div);
    });

    function getDistance(lat1, lon1, lat2, lon2) {
      var R = 6371;
      var dLat = (lat2-lat1)*Math.PI/180;
      var dLon = (lon2-lon1)*Math.PI/180;
      var a=Math.sin(dLat/2)**2 + Math.cos(lat1*Math.PI/180)*Math.cos(lat2*Math.PI/180)*Math.sin(dLon/2)**2;
      var c=2*Math.atan2(Math.sqrt(a),Math.sqrt(1-a));
      return R*c;
    }

    function updateNearestResponders(lat, lng){
      var distances = responders.map(r=>({...r, distance:getDistance(lat,lng,r.coords[0],r.coords[1])}));
      distances.sort((a,b)=>a.distance-b.distance);
      var top3 = distances.slice(0,3);
      document.getElementById("responderList").innerHTML="";
      top3.forEach(function(r){
        var div=document.createElement("div");
        div.className="responder";
        div.innerHTML=`<b>GeoTag #${r.id}</b><br>Name: ${r.name}<br>Mobile: ${r.mobile}<br>Distance: ${r.distance.toFixed(2)} km`;
        div.onclick=function(){
          map.setView(r.coords,16); markers[r.id].openPopup();
          if(!selectedNumbers.includes(r.mobile)){
            selectedNumbers.push(r.mobile);
            document.getElementById("numbers").value=selectedNumbers.join(", ");
          }
        };
        document.getElementById("responderList").appendChild(div);
      });
    }

    // Messaging
    function updateMessage() {
      var selectedEmergency=document.getElementById("template").value;
      var selectedBarangay=document.getElementById("barangay").value;
      var street=document.getElementById("street").value;
      var landmark=document.getElementById("landmark").value;
      var callback=document.getElementById("callback").value;
      var msg="";
      if(selectedEmergency) msg+=selectedEmergency+"\n";
      if(selectedBarangay) msg+="LOCATION: "+selectedBarangay+", Muntinlupa\n";
      if(street) msg+="STREET: "+street+"\n";
      if(landmark) msg+="NEAREST LANDMARK: "+landmark+"\n";
      if(callback) msg+="CALLBACK NUMBER: "+callback;
      document.getElementById("message").value=msg.trim();
    }

    document.getElementById("template").onchange=updateMessage;
    document.getElementById("barangay").onchange=updateMessage;
    document.getElementById("street").oninput=updateMessage;
    document.getElementById("landmark").oninput=updateMessage;
    document.getElementById("callback").oninput=updateMessage;

    // Stats
    let policeCount=0, barangayStats={}, totalResponded=0;
    document.getElementById("sendBtn").onclick=function(){
      var msg=document.getElementById("message").value;
      var selectedEmergency=document.getElementById("template").value;
      var selectedBarangay=document.getElementById("barangay").value;

      if(selectedNumbers.length===0 || msg.trim()===""){
        alert("Please select at least one responder and enter a message.");
        return;
      }

      if(selectedEmergency.includes("POLICE")) policeCount++;
      if(selectedBarangay){
        barangayStats[selectedBarangay]=(barangayStats[selectedBarangay]||0)+1;
      }
      totalResponded++;

      document.getElementById("policeCount").innerText=policeCount;
      document.getElementById("barangayCount").innerHTML=Object.keys(barangayStats).map(b=>b+": "+barangayStats[b]).join("<br>");
      document.getElementById("totalResponded").innerText=totalResponded;

      alert("Message sent to: "+selectedNumbers.join(", ")+"\n\n"+msg);
    };
  </script>
</body>
</html>
