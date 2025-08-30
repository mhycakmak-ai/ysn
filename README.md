# ysn
grdyo
// App.js
// BLE tarama + Konum + Sensör + 3D görselleştirme
// Not: Bu kod Expo bare workflow veya EAS Build ile derlenmeli
import React, { useEffect, useRef, useState } from 'react';
import { View, Text, Button, StyleSheet } from 'react-native';
import { BleManager } from 'react-native-ble-plx';
import * as Location from 'expo-location';
import { Accelerometer, Gyroscope } from 'expo-sensors';
import { GLView } from 'expo-gl';
import { Renderer } from 'expo-three';
import * as THREE from 'three';

export default function App() {
  const managerRef = useRef(null);
  const [devices, setDevices] = useState([]);
  const [location, setLocation] = useState(null);
  const [accel, setAccel] = useState({ x:0, y:0, z:0 });
  const [gyro, setGyro] = useState({ x:0, y:0, z:0 });
  const deviceMapRef = useRef(new Map());

  useEffect(() => {
    const m = new BleManager();
    managerRef.current = m;
    return () => { m.destroy(); };
  }, []);

  useEffect(() => {
    Accelerometer.setUpdateInterval(100);
    Gyroscope.setUpdateInterval(100);
    const aSub = Accelerometer.addListener(d => setAccel(d));
    const gSub = Gyroscope.addListener(d => setGyro(d));
    return () => { aSub.remove(); gSub.remove(); };
  }, []);

  useEffect(() => {
    (async () => {
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status === 'granted') {
        const loc = await Location.getCurrentPositionAsync({});
        setLocation(loc.coords);
        await Location.watchPositionAsync(
          { accuracy: Location.Accuracy.Highest, distanceInterval:1 },
          pos => setLocation(pos.coords)
        );
      } else console.warn('Location permission denied');
    })();
  }, []);

  const startScan = () => {
    const manager = managerRef.current;
    if (!manager) return;
    try { manager.stopDeviceScan(); } catch(e){}
    manager.startDeviceScan(null, { allowDuplicates:true }, (error, device) => {
      if (error || !device) return;
      const now = Date.now();
      const entry = { id: device.id, name: device.name||'unknown', rssi:device.rssi, lastSeen:now };
      deviceMapRef.current.set(device.id, entry);
      const arr = Array.from(deviceMapRef.current.values()).filter(d=>now-d.lastSeen<60000).slice(0,50);
      setDevices(arr);
    });
  };

  const stopScan = () => { managerRef.current?.stopDeviceScan(); };

  const onContextCreate = async (gl) => {
    const { drawingBufferWidth: width, drawingBufferHeight: height } = gl;
    const renderer = new Renderer({ gl });
    renderer.setSize(width, height);
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x111111);
    const camera = new THREE.PerspectiveCamera(70, width/height, 0.01, 1000);
    camera.position.z = 5;
    const light = new THREE.DirectionalLight(0xffffff,1);
    light.position.set(0,5,10).normalize();
    scene.add(light);

    const phoneMesh = new THREE.Mesh(
      new THREE.BoxGeometry(0.3,0.5,0.05),
      new THREE.MeshStandardMaterial({ color:0x00ff00 })
    );
    scene.add(phoneMesh);

    const btGroup = new THREE.Group();
    scene.add(btGroup);

    const rssiToDistance = rssi => Math.max(0.2, Math.min(20, Math.pow(10, (-59 - rssi)/20)));

    const updateBTObjects = () => {
      const map = new Map();
      btGroup.children.forEach(c=>map.set(c.userData.id,c));
      devices.forEach((d,i)=>{
        let obj = map.get(d.id);
        const dist = rssiToDistance(d.rssi);
        const angle = (i/Math.max(1,devices.length))*Math.PI*2;
        const x = Math.cos(angle)*dist;
        const y = Math.sin(angle)*dist*0.6;
        const z = ((d.rssi)?Math.min(10,-d.rssi/10):1)*0.2;
        if(!obj){
          const g = new THREE.SphereGeometry(0.15,12,12);
          const m = new THREE.MeshStandardMaterial({color:0x3399ff});
          obj = new THREE.Mesh(g,m);
          obj.userData={id:d.id};
          btGroup.add(obj);
        }
        obj.position.set(x,y,z);
        obj.scale.setScalar(1+(-d.rssi/100));
      });
      const ids = new Set(devices.map(d=>d.id));
      btGroup.children.slice().forEach(c=>{if(!ids.has(c.userData.id)) btGroup.remove(c);});
    };

    const animate = () => {
      requestAnimationFrame(animate);
      phoneMesh.rotation.x = accel.x*2;
      phoneMesh.rotation.y = accel.y*2;
      phoneMesh.rotation.z = gyro.z*0.5;
      updateBTObjects();
      camera.position.x = Math.sin(Date.now()*0.0002)*3;
      camera.lookAt(scene.position);
      renderer.render(scene,camera);
      gl.endFrameEXP();
    };
    animate();
  };

  return (
    <View style={{flex:1,paddingTop:40,backgroundColor:'#000'}}>
      <Text style={{color:'#fff',textAlign:'center',fontSize:18}}>3D BLE + Sensor Visualizer</Text>
      <View style={{flexDirection:'row',justifyContent:'space-around',marginVertical:8}}>
        <Button title="Start Scan" onPress={startScan}/>
        <Button title="Stop Scan" onPress={stopScan}/>
      </View>
      <View style={{padding:8,backgroundColor:'#111'}}>
        <Text>Location: {location?`${location.latitude.toFixed(5)},${location.longitude.toFixed(5)}`:'—'}</Text>
        <Text>Devices: {devices.length}</Text>
        <Text>Accel: {accel.x.toFixed(2)},{accel.y.toFixed(2)},{accel.z.toFixed(2)}</Text>
      </View>
      <GLView style={{flex:1,margin:8,borderRadius:8,overflow:'hidden'}} onContextCreate={onContextCreate}/>
    </View>
  );
}
