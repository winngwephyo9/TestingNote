import {
    Color,
    ColorManagement,
    DefaultLoadingManager,
    FileLoader,
    FrontSide,
    Loader,
    LoaderUtils,
    MeshStandardMaterial, // <-- CHANGE: Import MeshStandardMaterial
    RepeatWrapping,
    TextureLoader,
    Vector2,
    SRGBColorSpace
} from 'three';

/**
 * A loader for the MTL format.
 * ... (rest of the comments) ...
 */
class MTLLoader extends Loader {

    constructor(manager) {
        super(manager);
    }

    load(url, onLoad, onProgress, onError) {
        // ... (this function remains the same)
    }

    setMaterialOptions(value) {
        // ... (this function remains the same)
    }

    parse(text, path) {
        // ... (this function remains the same)
    }

}

class MaterialCreator {

    constructor(baseUrl = '', options = {}) {
        this.baseUrl = baseUrl;
        this.options = options;
        // ... (rest of constructor remains the same)
    }

    setCrossOrigin(value) { /* ... */ }
    setManager(value) { /* ... */ }
    setMaterials(materialsInfo) { /* ... */ }
    convert(materialsInfo) { /* ... */ }
    preload() { /* ... */ }
    getIndex(materialName) { /* ... */ }
    getAsArray() { /* ... */ }
    create(materialName) { /* ... */ }

    // =================================================================
    // --- START OF MODIFIED SECTION ---
    // =================================================================
    createMaterial_(materialName) {

        // Create material

        const scope = this;
        const mat = this.materialsInfo[materialName];
        const params = {
            name: materialName,
            side: this.side
        };

        function resolveURL(baseUrl, url) {
            if (typeof url !== 'string' || url === '') return '';
            if (/^https?:\/\//i.test(url)) return url;
            return baseUrl + url;
        }

        function setMapForType(mapType, value) {
            if (params[mapType]) return;
            const texParams = scope.getTextureParams(value, params);
            const map = scope.loadTexture(resolveURL(scope.baseUrl, texParams.url));
            map.repeat.copy(texParams.scale);
            map.offset.copy(texParams.offset);
            map.wrapS = scope.wrap;
            map.wrapT = scope.wrap;
            if (mapType === 'map' || mapType === 'emissiveMap') {
                map.colorSpace = SRGBColorSpace;
            }
            params[mapType] = map;
        }

        for (const prop in mat) {
            const value = mat[prop];
            let n;
            if (value === '') continue;

            switch (prop.toLowerCase()) {
                // Ns is material specular exponent
                case 'kd':
                    // Diffuse color becomes main color
                    params.color = ColorManagement.colorSpaceToWorking(new Color().fromArray(value), SRGBColorSpace);
                    break;
                case 'ks':
                    // Specular color can be roughly mapped to metalness or specular intensity
                    // For a non-metallic workflow, we can ignore this or map it to specularColor if needed.
                    // For a metallic workflow, this is more complex. Let's keep it simple.
                    // params.specular = new Color().fromArray( value ); // Not a direct property of MeshStandardMaterial
                    break;
                case 'ke':
                    // Emissive color
                    params.emissive = ColorManagement.colorSpaceToWorking(new Color().fromArray(value), SRGBColorSpace);
                    break;
                case 'map_kd':
                    setMapForType('map', value);
                    break;
                case 'map_ks':
                    // Can be mapped to roughnessMap or metalnessMap depending on workflow
                    setMapForType('roughnessMap', value); // A common rough approximation
                    break;
                case 'map_ke':
                    setMapForType('emissiveMap', value);
                    break;
                case 'norm':
                    setMapForType('normalMap', value);
                    break;
                case 'map_bump':
                case 'bump':
                    setMapForType('bumpMap', value);
                    break;
                case 'map_d':
                    setMapForType('alphaMap', value);
                    params.transparent = true;
                    break;
                case 'ns':
                    // Shininess can be inverted to create roughness.
                    // High shininess (e.g., 1000) -> low roughness (e.g., 0).
                    // Low shininess (e.g., 10) -> high roughness (e.g., 1).
                    params.roughness = 1 - (parseFloat(value) / 1000.0);
                    params.roughness = Math.max(0.0, Math.min(1.0, params.roughness)); // Clamp between 0 and 1
                    break;
                case 'd':
                    n = parseFloat(value);
                    if (n < 1) {
                        params.opacity = n;
                        params.transparent = true;
                    }
                    break;
                case 'tr':
                    n = parseFloat(value);
                    if (this.options && this.options.invertTrProperty) n = 1 - n;
                    if (n > 0) {
                        params.opacity = 1 - n;
                        params.transparent = true;
                    }
                    break;
                default:
                    break;
            }
        }

        // Set some defaults for PBR material if not defined
        if (params.roughness === undefined) params.roughness = 0.8;
        if (params.metalness === undefined) params.metalness = 0.1;

        // *** THE MAIN CHANGE IS THIS LINE ***
        this.materials[materialName] = new MeshStandardMaterial(params);
        return this.materials[materialName];
    }
    // =================================================================
    // --- END OF MODIFIED SECTION ---
    // =================================================================

    getTextureParams(value, matParams) {
        // ... (this function remains the same)
    }

    loadTexture(url, mapping, onLoad, onProgress, onError) {
        // ... (this function remains the same)
    }
}

export { MTLLoader };
