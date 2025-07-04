// Find this function in your MTLLoader.js and replace it entirely.

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
        // Correctly set color space for textures that affect color
        if (mapType === 'map' || mapType === 'emissiveMap') {
            // This property has existed for longer than .setColorSpace() on Color
            map.colorSpace = SRGBColorSpace;
        }
        params[mapType] = map;
    }

    for (const prop in mat) {
        const value = mat[prop];
        let n;
        if (value === '') continue;

        switch (prop.toLowerCase()) {
            case 'kd':
                // *** THE FIX IS HERE (COMPATIBLE VERSION) ***
                // Create a new color, set its RGB from the array,
                // then manually convert it from sRGB to the linear working space.
                params.color = new Color().fromArray(value).convertSRGBToLinear();
                break;
            case 'ks':
                // Specular color (rarely used in simple PBR)
                // params.specular = new Color().fromArray(value).convertSRGBToLinear();
                break;
            case 'ke':
                // Emissive color
                params.emissive = new Color().fromArray(value).convertSRGBToLinear();
                break;
            case 'map_kd':
                setMapForType('map', value);
                break;
            case 'map_ks':
                setMapForType('roughnessMap', value);
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
                params.roughness = 1.0 - (parseFloat(value) / 1000.0);
                params.roughness = Math.max(0.0, Math.min(1.0, params.roughness));
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

    if (params.roughness === undefined) params.roughness = 0.8;
    if (params.metalness === undefined) params.metalness = 0.1;

    this.materials[materialName] = new MeshStandardMaterial(params);
    return this.materials[materialName];
}
