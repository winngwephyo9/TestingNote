const scene = new THREE.Scene();

// --- NEW BACKGROUND AND GROUND ---
// Set a light, slightly blue sky color for the background
scene.background = new THREE.Color(0xd8e3ee); // A soft, light blue-gray

// Add a GridHelper to act as the ground plane
const grid_size = 500; // The overall size of the grid
const grid_divisions = 50; // The number of divisions
const grid_color_center = 0x888888; // Color of the center lines (X and Z axes)
const grid_color_grid = 0xcccccc; // Color of the other grid lines

const gridHelper = new THREE.GridHelper(grid_size, grid_divisions, grid_color_center, grid_color_grid);
scene.add(gridHelper);



<img width="989" height="707" alt="image" src="https://github.com/user-attachments/assets/c887bd8b-e3e0-4592-a942-8b59b931ce00" />
