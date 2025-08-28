import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';

// --- UI要素とグローバル変数の取得 ---
const loaderContainer = document.getElementById('loader-container');
const loaderTextElement = document.getElementById('loader-text');
const modelTreePanel = document.getElementById('modelTreePanel');
const modelTreeList = document.getElementById('modelTreeList');
const modelTreeSearch = document.getElementById('modelTreeSearch');
const closeModelTreeBtn = document.getElementById('closeModelTreeBtn');
const objectInfoPanel = document.getElementById('objectInfo');
const modelSelector = document.getElementById('model-selector');
const viewerContainer = document.getElementById('viewer-container');
const toggleUiButton = document.getElementById('toggle-ui-button');

let parsedWSCenID = "";
let parsedPJNo = "";
let loadedObjectModelRoot = null;
let selectedObjectOrGroup = null;
const originalMeshMaterials = new Map();
const originalObjectPropertiesForIsolate = new Map();
let isIsolateModeActive = false;
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xa0c4ff);
const elementIdDataMap = new Map();
const BOX_MAIN_FOLDER_ID = "332324771912";
var CSRF_TOKEN = $('meta[name="csrf-token"]').attr('content');

// --- Web Workerの初期化 ---
const objWorker = new Worker('js/obj-loader-worker.js', { type: 'module' });

// --- シーンのセットアップ ---
const scene = new THREE.Scene();
const backgroundGeometry = new THREE.PlaneGeometry(2, 2, 1, 1);
const backgroundMaterial = new THREE.ShaderMaterial({
    vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = vec4(position.xy, 1.0, 1.0); }`,
    fragmentShader: `uniform vec3 topColor; uniform vec3 bottomColor; varying vec2 vUv; void main() { gl_FragColor = vec4(mix(bottomColor, topColor, vUv.y), 1.0); }`,
    uniforms: { topColor: { value: new THREE.Color(0xd8e3ee) }, bottomColor: { value: new THREE.Color(0xf0f0f0) } },
    depthWrite: false
});
const backgroundMesh = new THREE.Mesh(backgroundGeometry, backgroundMaterial);
backgroundMesh.renderOrder = -1;
scene.add(backgroundMesh);
const camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.set(10, 10, 10);
const renderer = new THREE.WebGLRenderer({ antialias: true });
viewerContainer.appendChild(renderer.domElement);
const ambientLight = new THREE.AmbientLight(0x606060, 2);
scene.add(ambientLight);
const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 2.5);
directionalLight.position.set(50, 100, 75);
scene.add(directionalLight);
const hemiLight = new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 1.5);
hemiLight.position.set(0, 50, 0);
scene.add(hemiLight);
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// --- アプリケーションの初期化 ---
$(document).ready(function () {
    var login_user_id = $("#hidLoginID").val();
    var img_src = "/DL_DWH.png";
    var url = "DLDWH/objviewer";
    var content_name = "OBJビューア";
    recordAccessHistory(login_user_id, img_src, url, content_name);
    onWindowResize();
    populateProjectDropdown();
    animate();
});

async function populateProjectDropdown() {
    try {
        const projects = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getProjectList",
            data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID },
        });
        modelSelector.innerHTML = '';
        if (projects.includes("no_token")) {
            alert("BOXにログインされていないためobjファイルを取得きませんでした。");
        } else if (projects && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            loadModel(projects[2].id, projects[2].name);
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
    }
}

// ワーカーとの通信をPromiseでラップするヘルパー関数
function processGeometryWithWorker(combinedObjContent, mtlContent) {
    return new Promise((resolve, reject) => {
        const messageHandler = (event) => {
            objWorker.removeEventListener('message', messageHandler);
            objWorker.removeEventListener('error', errorHandler);
            if (event.data.error) {
                reject(new Error(event.data.error));
            } else {
                resolve(event.data);
            }
        };
        const errorHandler = (error) => {
            objWorker.removeEventListener('message', messageHandler);
            objWorker.removeEventListener('error', errorHandler);
            reject(error);
        };
        objWorker.addEventListener('message', messageHandler);
        objWorker.addEventListener('error', errorHandler);
        objWorker.postMessage({ combinedObjContent, mtlContent });
    });
}

async function loadModel(projectFolderId, projectName) {
    // 0. シーンと状態のリセット
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    parsedWSCenID = "";
    parsedPJNo = "";
    updateInfoPanel();
    
    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `ファイルリストを取得中: ${projectName}...`;

        // 1. ファイルリストとダウンロードURLを取得
        const fileList = await $.ajax({ type: "post", url: url_prefix + "/box/getObjList", data: { _token: CSRF_TOKEN, folderId: projectFolderId } });
        if (!fileList || !fileList.mtl || !fileList.objs || fileList.objs.length === 0) throw new Error("ファイルリストが不完全です。");

        const mtlFileInfo = fileList.mtl;
        const objFileInfoList = fileList.objs;
        const allFileIds = [mtlFileInfo.id, ...objFileInfoList.map(f => f.id)];
        const downloadUrlMap = {};
        const batchSize = 900;
        for (let i = 0; i < allFileIds.length; i += batchSize) {
            const batch = allFileIds.slice(i, i + batchSize);
            if (loaderTextElement) loaderTextElement.textContent = `ダウンロード準備中... (${i + batch.length}/${allFileIds.length})`;
            const batchUrlMap = await $.ajax({ type: "post", url: url_prefix + "/box/getDownloadUrls", data: { _token: window.CSRF_TOKEN, fileIds: batch } });
            Object.assign(downloadUrlMap, batchUrlMap);
        }

        // 2. 全てのファイルコンテンツをテキストとしてダウンロード
        const mtlUrl = downloadUrlMap[mtlFileInfo.id];
        const mtlContent = await fetch(mtlUrl).then(res => res.text());
        const allObjContents = await downloadAllObjs(objFileInfoList, downloadUrlMap);

        if (allObjContents.length > 0) {
            const headerData = await parseObjHeader(allObjContents[0].content);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
                updateInfoPanel();
            }
        }
        
        // 3. 全てのOBJファイルの中身を一つの巨大な文字列に結合します。
        if (loaderTextElement) loaderTextElement.textContent = 'ジオメトリファイルを結合中...';
        const combinedObjContent = allObjContents.map(obj => obj.content).join('\n');

        // 4. ワーカーに単一のパースジョブを依頼します
        if (loaderTextElement) loaderTextElement.textContent = `ジオメトリをバックグラウンドで処理中...`;
        const modelJson = await processGeometryWithWorker(combinedObjContent, mtlContent);

        console.log("Main: ワーカーから処理済みモデルを受信しました。");
        if (loaderTextElement) loaderTextElement.textContent = `シーンを最終処理中...`;

        // 5. ワーカーからのJSONデータでモデルを再構築
        const loader = new THREE.ObjectLoader();
        loadedObjectModelRoot = loader.parse(modelJson);
        if (!loadedObjectModelRoot || loadedObjectModelRoot.children.length === 0) throw new Error("モデルの処理後、中身が空です。");

        // 6. シーンの最終処理
        const box = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const center = box.getCenter(new THREE.Vector3());
        loadedObjectModelRoot.position.sub(center);
        const size = box.getSize(new THREE.Vector3());
        const maxDim = Math.max(size.x, size.y, size.z);
        if (maxDim > 0) {
            const scale = 150 / maxDim;
            loadedObjectModelRoot.scale.set(scale, scale, scale);
        }
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        const allIds = loadedObjectModelRoot.children.map(child => {
            if (!child.name) return null;
            const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
            return splitIndex > 0 ? child.name.substring(splitIndex + 1) : null;
        }).filter(Boolean);
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);

        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        if (loaderContainer) loaderContainer.style.display = 'none';

    } catch (error) {
        console.error(`モデルのロードに失敗しました:`, error);
        if (loaderTextElement) loaderTextElement.textContent = `エラーが発生しました。コンソールを確認してください。`;
    }
}

async function downloadAllObjs(objFileInfoList, downloadUrlMap) {
    if (loaderTextElement) loaderTextElement.textContent = `ジオメトリをダウンロード中 (0/${objFileInfoList.length})...`;
    const CONCURRENT_DOWNLOADS = 10;
    const allObjContents = [];
    let downloadedCount = 0;
    const downloadQueue = [...objFileInfoList];

    const downloadBatch = async () => {
        const promises = [];
        while (downloadQueue.length > 0 && promises.length < CONCURRENT_DOWNLOADS) {
            const objInfo = downloadQueue.shift();
            const objUrl = downloadUrlMap[objInfo.id];
            if (objInfo && objUrl) {
                promises.push(
                    fetch(objUrl)
                        .then(res => res.ok ? res.text() : Promise.reject(new Error(res.statusText)))
                        .then(content => {
                            downloadedCount++;
                            if (loaderTextElement) loaderTextElement.textContent = `ジオメトリをダウンロード中 (${downloadedCount}/${objFileInfoList.length})...`;
                            return { content, info: objInfo };
                        })
                        .catch(err => {
                            console.warn(`${objInfo.name}のダウンロードに失敗:`, err);
                            downloadedCount++;
                            if (loaderTextElement) loaderTextElement.textContent = `ジオメトリをダウンロード中 (${downloadedCount}/${objFileInfoList.length})...`;
                            return null;
                        })
                );
            }
        }
        const results = await Promise.all(promises);
        allObjContents.push(...results.filter(Boolean));
        if (downloadQueue.length > 0) await downloadBatch();
    };
    await downloadBatch();
    return allObjContents;
}

async function parseObjHeader(objContent) {
    try {
        const lines = objContent.split(/\r?\n/);
        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) return { wscenId: pattern1Match[1], pjNo: pattern1Match[2] };
                if (content.includes("ワークシェアリングされてない")) return { wscenId: "", pjNo: "" };
            }
        }
    } catch (error) { console.error("Error parsing OBJ header:", error); }
    return { wscenId: null, pjNo: null };
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    if (!wscenId || allElementIds.length === 0) return;
    const batchSize = 900;
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        if (loaderTextElement) loaderTextElement.textContent = `カテゴリ情報を取得中... (${i + batch.length}/${allElementIds.length})`;
        try {
            const data = await $.ajax({ type: "post", url: url_prefix + "/DLDWH/getDatas", data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementIds: batch } });
            for (const elementId in data) {
                elementIdDataMap.set(elementId, data[elementId]);
            }
        } catch (err) {
            console.error(`カテゴリ情報のバッチ取得エラー:`, err);
        }
    }
}

async function buildAndPopulateCategorizedTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;
    if (loaderTextElement) loaderTextElement.textContent = "モデルツリーを構築中...";
    await new Promise(resolve => setTimeout(resolve, 50));

    const categorizedObjects = {};
    loadedObjectModelRoot.children.forEach(child => {
        if (child.isGroup && child.parent === loadedObjectModelRoot) {
            let rawName = child.name;
            let displayId = null;
            const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
            if (splitIndex > 0) displayId = rawName.substring(splitIndex + 1);
            let category = "カテゴリー無し";
            if (displayId && elementIdDataMap.has(displayId)) {
                category = elementIdDataMap.get(displayId)['カテゴリー名'] || "カテゴリー無し";
            } else if (!displayId) {
                category = "名称未分類";
            }
            if (!categorizedObjects[category]) categorizedObjects[category] = [];
            categorizedObjects[category].push(child);
        }
    });
    
    modelTreeList.innerHTML = '';
    const fragment = document.createDocumentFragment();
    Object.keys(categorizedObjects).sort().forEach(categoryName => {
        fragment.appendChild(createCategoryNode(categoryName, categorizedObjects[categoryName]));
    });
    modelTreeList.appendChild(fragment);
    if (modelTreePanel) modelTreePanel.style.display = 'block';
}

function createCategoryNode(categoryName, objectsInCategory) {
    const categoryLi = document.createElement('li');
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.fontWeight = 'bold';
    const toggler = document.createElement('span');
    toggler.className = 'toggler';
    toggler.textContent = '▼';
    itemContent.appendChild(toggler);
    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = `${categoryName} (${objectsInCategory.length})`;
    itemContent.appendChild(nameSpan);
    const subList = document.createElement('ul');
    toggler.addEventListener('click', (e) => {
        e.stopPropagation();
        const isCollapsed = subList.style.display === 'none';
        subList.style.display = isCollapsed ? 'block' : 'none';
        toggler.textContent = isCollapsed ? '▼' : '▶';
    });
    categoryLi.appendChild(itemContent);
    categoryLi.appendChild(subList);
    objectsInCategory.forEach(object => createObjectNode(object, subList, 1));
    return categoryLi;
}

function createObjectNode(object, parentULElement, depth) {
    const listItem = document.createElement('li');
    listItem.dataset.uuid = object.uuid;
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.paddingLeft = `${depth * 15 + 10}px`;
    const toggler = document.createElement('span');
    toggler.className = 'toggler empty-toggler';
    toggler.innerHTML = '&nbsp;';
    itemContent.appendChild(toggler);
    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = object.name;
    nameSpan.title = object.name;
    itemContent.appendChild(nameSpan);
    const visibilityToggle = document.createElement('span');
    visibilityToggle.className = 'visibility-toggle visible-icon';
    visibilityToggle.title = 'Hide';
    itemContent.appendChild(visibilityToggle);
    listItem.appendChild(itemContent);
    parentULElement.appendChild(listItem);
    itemContent.addEventListener('click', () => handleSelection(object));
    visibilityToggle.addEventListener('click', (e) => {
        e.stopPropagation();
        object.visible = !object.visible;
        visibilityToggle.classList.toggle('visible-icon', object.visible);
        visibilityToggle.classList.toggle('hidden-icon', !object.visible);
        visibilityToggle.title = object.visible ? 'Hide' : 'Show';
        if (!object.visible && selectedObjectOrGroup && selectedObjectOrGroup.uuid === object.uuid) {
            handleSelection(null);
        }
    });
}

function frameObject(objectToFrame) {
    const box = new THREE.Box3().setFromObject(objectToFrame);
    if (box.isEmpty()) return;
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const distance = (sphere.radius / Math.sin(fovInRadians / 2)) * 1.3;
    const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
    
    camera.position.copy(center).addScaledVector(cameraDirection, distance);
    camera.lookAt(center);
    controls.target.copy(center);
    controls.update();
}

function handleSelection(target) {
    removeAllHighlights();
    deIsolateAllObjects();
    let newSelection = null;
    if (target && (!selectedObjectOrGroup || selectedObjectOrGroup.uuid !== target.uuid)) {
        newSelection = target;
    }
    selectedObjectOrGroup = newSelection;
    if (selectedObjectOrGroup) {
        applyHighlight(selectedObjectOrGroup, highlightColorSingle);
        zoomToAndIsolate(selectedObjectOrGroup);
    }
    document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
    if (selectedObjectOrGroup) {
        const treeItemDiv = document.querySelector(`#modelTreePanel li[data-uuid="${selectedObjectOrGroup.uuid}"] .tree-item`);
        if (treeItemDiv) treeItemDiv.classList.add('selected');
    }
    updateInfoPanel();
}

const applyHighlight = (target, color) => {
    if (!target) return;
    target.traverse(child => {
        if (child.isMesh) {
            if (!originalMeshMaterials.has(child.uuid)) {
                originalMeshMaterials.set(child.uuid, child.material);
            }
            if (Array.isArray(child.material)) {
                child.material = child.material.map(mat => {
                    const newMat = mat.clone();
                    newMat.color.set(color);
                    return newMat;
                });
            } else {
                const newMat = child.material.clone();
                newMat.color.set(color);
                child.material = newMat;
            }
        }
    });
};

const removeAllHighlights = () => {
    originalMeshMaterials.forEach((originalMaterial, uuid) => {
        const mesh = scene.getObjectByProperty('uuid', uuid);
        if (mesh) mesh.material = originalMaterial;
    });
    originalMeshMaterials.clear();
};

function zoomToAndIsolate(targetObject) {
    if (!targetObject) return;
    deIsolateAllObjects();
    isIsolateModeActive = true;
    const box = new THREE.Box3().setFromObject(targetObject);
    if (box.isEmpty()) { isIsolateModeActive = false; return; }
    
    frameObject(targetObject);

    loadedObjectModelRoot.traverse((object) => {
        if (object.isMesh) {
            let isPartOfSelected = false;
            let temp = object;
            while(temp) {
                if (temp === targetObject) {
                    isPartOfSelected = true;
                    break;
                }
                temp = temp.parent;
            }

            if (!isPartOfSelected) {
                if (!originalObjectPropertiesForIsolate.has(object.uuid)) {
                    originalObjectPropertiesForIsolate.set(object.uuid, { material: object.material, visible: object.visible });
                }
                if (object.visible) {
                    const materials = Array.isArray(object.material) ? object.material : [object.material];
                    const newMaterials = materials.map(mat => {
                        const newMat = mat.clone();
                        newMat.transparent = true;
                        newMat.opacity = 0.1;
                        return newMat;
                    });
                    object.material = Array.isArray(object.material) ? newMaterials : newMaterials[0];
                }
            }
        }
    });
}

function deIsolateAllObjects() {
    if (!isIsolateModeActive) return;
    originalObjectPropertiesForIsolate.forEach((props, uuid) => {
        const object = scene.getObjectByProperty('uuid', uuid);
        if (object && object.isMesh) {
            object.material = props.material;
            object.visible = props.visible;
        }
    });
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
}

function updateInfoPanel() {
    let headerInfo = `WSCenID: ${parsedWSCenID || "N/A"}\nPJNo: ${parsedPJNo || "N/A"}\n----\n`;
    if (parsedWSCenID === "" && parsedPJNo === "") headerInfo = `WSCenID: \nPJNo: \n----\n`;
    if (selectedObjectOrGroup) {
        let rawName = selectedObjectOrGroup.name || "Unnamed";
        let displayId = "N/A";
        const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
        if (splitIndex > 0) displayId = rawName.substring(splitIndex + 1);
        let selectionInfo = `ID: ${displayId}\n`;
        if (displayId && elementIdDataMap.has(displayId)) {
            const data = elementIdDataMap.get(displayId);
            selectionInfo += `カテゴリー名: ${data['カテゴリー名'] || "N/A"}\nファミリ名: ${data['ファミリ名'] || "N/A"}`;
        }
        objectInfoPanel.textContent = headerInfo + selectionInfo;
    } else {
        objectInfoPanel.textContent = headerInfo + 'None Selected';
    }
}

window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot || event.target.closest('#modelTreePanel')) return;
    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);
    if (intersects.length > 0) {
        let current = intersects[0].object;
        while (current && current.parent !== loadedObjectModelRoot) {
            current = current.parent;
        }
        handleSelection(current);
    } else {
        handleSelection(null);
    }
});

if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });

if (modelTreeSearch) {
    modelTreeSearch.addEventListener('input', (e) => {
        const searchTerm = e.target.value.toLowerCase().trim();
        document.querySelectorAll('#modelTreeList li .group-name').forEach(nameSpan => {
            const li = nameSpan.closest('li');
            const isMatch = nameSpan.textContent.toLowerCase().includes(searchTerm);
            li.style.display = isMatch ? '' : 'none';
        });
    });
}

if (toggleUiButton) {
    toggleUiButton.addEventListener('click', () => {
        const isVisible = modelTreePanel.style.display !== 'none';
        if (modelTreePanel) modelTreePanel.style.display = isVisible ? 'none' : 'block';
        toggleUiButton.textContent = isVisible ? '📊' : '❌';
        toggleUiButton.title = isVisible ? "Show UI Panels" : "Hide UI Panels";
    });
}

function onWindowResize() {
    const { clientWidth, clientHeight } = viewerContainer;
    camera.aspect = clientWidth / clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(clientWidth, clientHeight);
}
window.addEventListener('resize', onWindowResize);

if (modelSelector) {
    modelSelector.addEventListener('change', (event) => {
        const selectedId = event.target.value;
        const selectedName = event.target.options[event.target.selectedIndex].text;
        loadModel(selectedId, selectedName);
    });
}

function animate() {
    requestAnimationFrame(animate);
    controls.update();
    // 黒い背景の問題を修正
    renderer.autoClear = false;
    renderer.render(scene, camera);
}
