import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/controls/OBJLoader.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0xcccccc); // Added a light gray background

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 20000); // Increased far plane
const renderer = new THREE.WebGLRenderer({ antialias: true }); // Added antialias
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const ambientLight = new THREE.AmbientLight(0x404040, 5); // Adjusted intensity
scene.add(ambientLight);
const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 3); // Adjusted intensity
directionalLight.position.set(10, 10, 10); // Adjusted light position for potentially larger scene
directionalLight.castShadow = true; // Optional: if you want shadows
scene.add(directionalLight);

// Helper light to see orientation better
const hemiLight = new THREE.HemisphereLight( 0xffffff, 0x444444, 2 );
hemiLight.position.set( 0, 20, 0 );
scene.add( hemiLight );

const geometry = new THREE.BoxGeometry();
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 }); // Changed cube color for distinction
const cube = new THREE.Mesh(geometry, material);
cube.name = "Demo Cube";
scene.add(cube);

camera.position.set(10, 10, 10); // Adjusted initial camera position
camera.lookAt(scene.position);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.1; // Adjusted damping
controls.screenSpacePanning = false;
controls.minDistance = 1;
controls.maxDistance = 10000; // Allow zooming out further

const objLoader = new OBJLoader();
// IMPORTANT: Place 'standard_testing.obj' and 'obj出力検証用_【標準書き出し】.mtl'
// in a folder named 'objFiles' next to your HTML file.
const objPath = './objFiles/standard_testing.obj'; // Corrected path

objLoader.load(
    objPath,
    (object) => {
        // Center and scale the object
        const box = new THREE.Box3().setFromObject(object);
        const center = box.getCenter(new THREE.Vector3());
        const size = box.getSize(new THREE.Vector3());

        // Move object to origin
        object.position.sub(center);

        // Scale object to a reasonable size (e.g., 10 units max dimension)
        const maxDim = Math.max(size.x, size.y, size.z);
        if (maxDim > 0) { // Avoid division by zero if object is empty or flat
            const desiredSize = 10;
            const scale = desiredSize / maxDim;
            object.scale.set(scale, scale, scale);
        }
        
        // Optional: Slightly move the scaled object if needed, e.g., next to the cube
        // object.position.x = 5; // Example offset

        scene.add(object);
        console.log("OBJ loaded successfully:", object);

        // Adjust controls target to the center of the loaded object
        const newBox = new THREE.Box3().setFromObject(object);
        const newCenter = newBox.getCenter(new THREE.Vector3());
        controls.target.copy(newCenter);
        camera.lookAt(newCenter); // Ensure camera looks at the new target
        controls.update();
    },
    (xhr) => {
        console.log((xhr.loaded / xhr.total * 100) + '% loaded');
    },
    (error) => {
        console.error('An error happened while loading the OBJ file:', error);
        const errorDiv = document.createElement('div');
        errorDiv.textContent = `Error loading OBJ: ${objPath}. Check console and file path/contents. MTL file might also be an issue.`;
        errorDiv.style.color = 'red';
        errorDiv.style.position = 'absolute';
        errorDiv.style.top = '30px';
        errorDiv.style.left = '10px';
        errorDiv.style.backgroundColor = 'white';
        errorDiv.style.padding = '10px';
        errorDiv.style.zIndex = "100";
        document.body.appendChild(errorDiv);
    }
);

let selectedObjects = [];
let isShiftDown = false;
const originalMaterials = new Map(); // To store original materials/colors

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

window.addEventListener('keydown', (event) => {
    if (event.key === 'Shift') isShiftDown = true;
});
window.addEventListener('keyup', (event) => {
    if (event.key === 'Shift') isShiftDown = false;
});

window.addEventListener('click', (event) => {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children, true); // true for recursive

    const highlightColorSingle = new THREE.Color(0xffff00); // Yellow for single select
    const highlightColorMulti = new THREE.Color(0xff0000); // Red for multi-select (Shift)

    // Helper to apply highlight
    const applyHighlight = (obj, color) => {
        if (!obj.isMesh || !obj.material) return;
        if (!originalMaterials.has(obj.uuid)) {
            if (Array.isArray(obj.material)) {
                originalMaterials.set(obj.uuid, obj.material.map(m => m.clone()));
            } else {
                originalMaterials.set(obj.uuid, obj.material.clone());
            }
        }
        if (Array.isArray(obj.material)) {
            obj.material.forEach(m => { if(m.color) m.color.set(color); else if (m.emissive) m.emissive.set(color); });
        } else {
            if (obj.material.color) obj.material.color.set(color);
            else if (obj.material.emissive) obj.material.emissive.set(color); // Fallback to emissive
        }
    };

    // Helper to remove highlight
    const removeHighlight = (obj) => {
        if (!obj.isMesh || !obj.material) return;
        if (originalMaterials.has(obj.uuid)) {
            const originalMat = originalMaterials.get(obj.uuid);
             // If the object material was an array, originalMat is an array of cloned materials
            if (Array.isArray(obj.material) && Array.isArray(originalMat)) {
                obj.material = originalMat.map(m => m.clone()); // Restore by cloning from stored clones
            } 
            // If the object material was single, originalMat is a single cloned material
            else if (!Array.isArray(obj.material) && !Array.isArray(originalMat) && originalMat) {
                 obj.material = originalMat.clone(); // Restore by cloning
            }
            // originalMaterials.delete(obj.uuid); // Keep it for re-highlighting without losing original
        }
    };

    if (intersects.length > 0) {
        const clickedObject = intersects[0].object;

        if (clickedObject.isMesh && clickedObject.material) { // Ensure it's a selectable mesh
            if (isShiftDown) {
                const index = selectedObjects.indexOf(clickedObject);
                if (index === -1) { // Not selected, add to selection
                    selectedObjects.push(clickedObject);
                    applyHighlight(clickedObject, highlightColorMulti);
                } else { // Already selected, remove from selection
                    removeHighlight(clickedObject);
                    selectedObjects.splice(index, 1);
                }
            } else {
                // Single selection
                // Deselect all previously selected objects
                selectedObjects.forEach(obj => removeHighlight(obj));
                originalMaterials.clear(); // Clear all stored for single selection mode to re-capture fresh state

                // Select the new one
                selectedObjects = [clickedObject];
                applyHighlight(clickedObject, highlightColorSingle);
            }
        }
    } else {
        // Clicked on empty space, deselect all
        selectedObjects.forEach(obj => removeHighlight(obj));
        selectedObjects = [];
        originalMaterials.clear(); // Clear all stored on full deselect
    }
    updateInfoPanel();
});

function updateInfoPanel() {
    const info = selectedObjects.map(obj => {
        const pos = obj.getWorldPosition(new THREE.Vector3());
        // Try to get a meaningful name, traversing up if needed
        let name = obj.name || (obj.parent ? obj.parent.name : "Unnamed");
        if (name === "" && obj.parent && obj.parent.parent) { // Potentially nested deeper
            name = obj.parent.parent.name || "Unnamed";
        }
        return `${name}\n  x: ${pos.x.toFixed(2)}, y: ${pos.y.toFixed(2)}, z: ${pos.z.toFixed(2)}`;
    }).join('\n\n');
    document.getElementById('objectInfo').textContent = info || 'None';
}

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

function animate() {
    requestAnimationFrame(animate);
    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;
    controls.update();
    renderer.render(scene, camera);
}

animate();
