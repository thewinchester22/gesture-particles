<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ZenParticles - Static Idle</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600&display=swap" rel="stylesheet">
    
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>

    <style>
      body { margin: 0; font-family: 'Inter', sans-serif; background: #000; overflow: hidden; }
      canvas { display: block; }
      @keyframes slide-in-right { from { transform: translateX(20px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
      .slide-in-from-right-10 { animation: slide-in-right 0.5s ease-out forwards; }
    </style>
    
    <script type="importmap">
    {
      "imports": {
        "react": "https://esm.sh/react@18.2.0",
        "react-dom/client": "https://esm.sh/react-dom@18.2.0/client",
        "@react-three/fiber": "https://esm.sh/@react-three/fiber@8.16.8",
        "@react-three/drei": "https://esm.sh/@react-three/drei@9.108.0",
        "three": "https://esm.sh/three@0.165.0",
        "lucide-react": "https://esm.sh/lucide-react@0.394.0"
      }
    }
    </script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  </head>
  <body>
    <div id="root"></div>

    <script type="text/babel" data-type="module" data-presets="react,typescript">
      import React, { useState, useEffect, useRef, useMemo, Suspense } from 'react';
      import { createRoot } from 'react-dom/client';
      import { Canvas, useFrame } from '@react-three/fiber';
      import { OrbitControls, Stars } from '@react-three/drei';
      import * as THREE from 'three';
      import { Heart, Flower, Globe, User, Sparkles, Zap, Power, Move } from 'lucide-react';

      // --- TYPES ---
      const ParticleShape = {
        HEART: 'HEART',
        FLOWER: 'FLOWER',
        SATURN: 'SATURN',
        BUDDHA: 'BUDDHA',
        FIREWORKS: 'FIREWORKS',
      };

      // --- PERLIN NOISE ---
      class Perlin {
        constructor() {
          this.permutation = [151,160,137,91,90,15,131,13,201,95,96,53,194,233,7,225,140,36,103,30,69,142,8,99,37,240,21,10,23,190,6,148,247,120,234,75,0,26,197,62,94,252,219,203,117,35,11,32,57,177,33,88,237,149,56,87,174,20,125,136,171,168,68,175,74,165,71,134,139,48,27,166,77,146,158,231,83,111,229,122,60,211,133,230,220,105,92,41,55,46,245,40,244,102,143,54,65,25,63,161,1,216,80,73,209,76,132,187,208,89,18,169,200,196,135,130,116,188,159,86,164,100,109,198,173,186,3,64,52,217,226,72,124,123,5,202,38,147,118,126,255,82,85,212,207,206,59,227,47,16,58,17,182,189,28,42,223,183,170,213,119,248,152,2,44,154,163,70,221,153,101,155,167,43,172,9,129,22,39,253,19,98,108,110,79,113,224,232,178,185,112,104,218,246,97,228,251,34,242,193,238,210,144,12,191,179,162,241,81,51,145,235,249,14,239,107,49,192,214,31,181,199,106,157,184,84,204,176,115,121,50,45,127,4,150,254,138,236,205,93,222,114,67,29,24,72,243,141,128,195,78,66,215,61,156,180];
          this.p = new Array(512);
          for (let i = 0; i < 256; i++) this.p[256 + i] = this.p[i] = this.permutation[i];
        }
        fade(t) { return t * t * t * (t * (t * 6 - 15) + 10); }
        lerp(t, a, b) { return a + t * (b - a); }
        grad(hash, x, y, z) {
          const h = hash & 15;
          const u = h < 8 ? x : y;
          const v = h < 4 ? y : h === 12 || h === 14 ? x : z;
          return ((h & 1) === 0 ? u : -u) + ((h & 2) === 0 ? v : -v);
        }
        noise(x, y, z) {
          const X = Math.floor(x) & 255, Y = Math.floor(y) & 255, Z = Math.floor(z) & 255;
          x -= Math.floor(x); y -= Math.floor(y); z -= Math.floor(z);
          const u = this.fade(x), v = this.fade(y), w = this.fade(z);
          const A = this.p[X] + Y, AA = this.p[A] + Z, AB = this.p[A + 1] + Z, B = this.p[X + 1] + Y, BA = this.p[B] + Z, BB = this.p[B + 1] + Z;
          return this.lerp(w, this.lerp(v, this.lerp(u, this.grad(this.p[AA], x, y, z), this.grad(this.p[BA], x - 1, y, z)), this.lerp(u, this.grad(this.p[AB], x, y - 1, z), this.grad(this.p[BB], x - 1, y - 1, z))), this.lerp(v, this.lerp(u, this.grad(this.p[AA + 1], x, y, z - 1), this.grad(this.p[BA + 1], x - 1, y, z - 1)), this.lerp(u, this.grad(this.p[AB + 1], x, y - 1, z - 1), this.grad(this.p[BB + 1], x - 1, y - 1, z - 1))));
        }
      }

      // --- SHAPE GENERATION ---
      const generateParticles = (shape, count) => {
        const positions = new Float32Array(count * 3);
        for (let i = 0; i < count; i++) {
          const i3 = i * 3;
          let x = 0, y = 0, z = 0;
          switch (shape) {
            case ParticleShape.HEART: {
              const t = Math.random() * Math.PI * 2;
              const r = Math.sqrt(Math.random()) * 2;
              const hx = 16 * Math.pow(Math.sin(t), 3);
              const hy = 13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t);
              x = hx * 0.1 * r; y = hy * 0.1 * r; z = (Math.random() - 0.5) * 2; 
              break;
            }
            case ParticleShape.FLOWER: {
              const theta = Math.random() * Math.PI * 2;
              const phi = Math.random() * Math.PI;
              const k = 4;
              const r = 2 + Math.cos(k * theta) * Math.sin(phi);
              x = r * Math.sin(phi) * Math.cos(theta);
              y = r * Math.sin(phi) * Math.sin(theta);
              z = r * Math.cos(phi) * 0.5;
              break;
            }
            case ParticleShape.SATURN: {
              const isRing = Math.random() > 0.6;
              if (isRing) {
                const angle = Math.random() * Math.PI * 2;
                const dist = 3 + Math.random() * 2.5;
                x = Math.cos(angle) * dist; z = Math.sin(angle) * dist; y = (Math.random() - 0.5) * 0.2;
                const tilt = Math.PI / 6;
                const yt = y * Math.cos(tilt) - z * Math.sin(tilt);
                const zt = y * Math.sin(tilt) + z * Math.cos(tilt);
                y = yt; z = zt;
              } else {
                const u = Math.random(), v = Math.random();
                const theta = 2 * Math.PI * u, phi = Math.acos(2 * v - 1), r = 1.8;
                x = r * Math.sin(phi) * Math.cos(theta); y = r * Math.sin(phi) * Math.sin(theta); z = r * Math.cos(phi);
              }
              break;
            }
            case ParticleShape.BUDDHA: {
              const part = Math.random();
              if (part < 0.4) {
                  const u = Math.random(), v = Math.random(), theta = 2 * Math.PI * u, phi = Math.acos(2 * v - 1);
                  x = 2.5 * Math.sin(phi) * Math.cos(theta); y = 1.0 * Math.sin(phi) * Math.sin(theta) - 1.5; z = 2.0 * Math.cos(phi);
              } else if (part < 0.8) {
                  const theta = Math.random() * Math.PI * 2, h = Math.random() * 2.5, r = 1.0 - (h * 0.1);
                  x = r * Math.cos(theta); y = h - 1.5; z = r * Math.sin(theta);
              } else {
                  const u = Math.random(), v = Math.random(), theta = 2 * Math.PI * u, phi = Math.acos(2 * v - 1), r = 0.8;
                  x = r * Math.sin(phi) * Math.cos(theta); y = r * Math.sin(phi) * Math.sin(theta) + 1.8; z = r * Math.cos(phi);
              }
              break;
            }
            case ParticleShape.FIREWORKS: {
               const u = Math.random(), v = Math.random(), theta = 2 * Math.PI * u, phi = Math.acos(2 * v - 1), r = Math.random() * 0.2;
               x = r * Math.sin(phi) * Math.cos(theta); y = r * Math.sin(phi) * Math.sin(theta); z = r * Math.cos(phi);
               break;
            }
          }
          positions[i3] = x; positions[i3 + 1] = y; positions[i3 + 2] = z;
        }
        return positions;
      };

      // --- COMPONENTS ---
      const ParticleSystem = ({ shape, color, count, interactionState }) => {
        const pointsRef = useRef(null);
        const materialRef = useRef(null);
        const perlin = useMemo(() => new Perlin(), []);
        const targetPositions = useMemo(() => generateParticles(shape, count), [shape, count]);
        const particlesRef = useRef({ positions: new Float32Array(count * 3), velocities: new Float32Array(count * 3) });
        
        // Store rotation target
        const rotationRef = useRef({ x: 0, y: 0 });

        useEffect(() => {
          particlesRef.current.positions.set(targetPositions);
          particlesRef.current.velocities.fill(0);
          if (pointsRef.current) {
              pointsRef.current.geometry.attributes.position.array.set(particlesRef.current.positions);
              pointsRef.current.geometry.attributes.position.needsUpdate = true;
          }
        }, [targetPositions, count]);

        useFrame((state) => {
          if (!pointsRef.current) return;
          const geometry = pointsRef.current.geometry;
          const time = state.clock.getElapsedTime();
          
          const tension = interactionState.tension; 
          
          // --- ROTATION LOGIC ---
          let targetRotX = 0;
          let targetRotY = 0;

          if (interactionState.isActive) {
             // Map Hand Position to Rotation
             targetRotY = (interactionState.x - 0.5) * 4; 
             targetRotX = (interactionState.y - 0.5) * 2; 
          } else {
             // IDLE: Return to Center (0,0) - Truly Static
             targetRotX = 0;
             targetRotY = 0;
          }

          // Smooth lerp for rotation
          rotationRef.current.x += (targetRotX - rotationRef.current.x) * 0.05;
          rotationRef.current.y += (targetRotY - rotationRef.current.y) * 0.05;
          
          pointsRef.current.rotation.y = rotationRef.current.y;
          pointsRef.current.rotation.x = rotationRef.current.x;

          // --- PARTICLE PHYSICS ---
          const { positions, velocities } = particlesRef.current;
          const targets = targetPositions;

          const currentScale = 1.5 - (tension * 1.0);
          
          // If inactive, drastically reduce noise speed for "Frozen" look
          const nSpeed = interactionState.isActive ? (0.3 + (tension * 3.7)) : 0.05; 
          const attraction = 0.015 + (tension * 0.135);
          const nFreq = 0.8 + (tension * 0.7); 
          const nAmp = 0.03 + (tension * 0.12);
          const damping = 0.92 - (tension * 0.32);

          for (let i = 0; i < count; i++) {
            const i3 = i * 3;
            const px = positions[i3], py = positions[i3 + 1], p
