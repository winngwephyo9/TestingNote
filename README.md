/**
 * WORKER SCRIPT (obj-loader-worker.js)
 * 最適化版：単一の結合済みOBJ文字列を受け取り、一度だけパース処理を行います。
 */

import * as THREE from './library/three.module.js';
import { OBJLoader } from './library/controls/OBJLoader.js';
import { MTLLoader } from './library/controls/MTLLoader.js';

const objLoader = new OBJLoader();
const mtlLoader = new MTLLoader();

self.onmessage = function (event) {
    const { combinedObjContent, mtlContent } = event.data;

    console.log("Worker: 結合されたデータを受信し、単一パース処理を開始します...");

    try {
        // 1. MTLコンテンツをパースしてマテリアルを作成します
        const materialsCreator = mtlLoader.parse(mtlContent);
        materialsCreator.preload();
        objLoader.setMaterials(materialsCreator);

        // 2. ***重要な最適化***
        // 巨大な単一のOBJ文字列を一度だけパースします。これにより処理が劇的に速くなります。
        const finalModel = objLoader.parse(combinedObjContent);

        if (!finalModel || finalModel.children.length === 0) {
            throw new Error("結合されたOBJ文字列のパース結果が空のモデルになりました。");
        }

        console.log("Worker: 単一パースが完了し、モデルを返送します。");

        // 3. 最終モデルをJSONに変換してメインスレッドに返します
        self.postMessage(finalModel.toJSON());

    } catch (error) {
        console.error("Worker Error:", error);
        self.postMessage({ error: error.message });
    }
};




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
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// --- アプリケーションの初期化 ---
$(document).ready(function () {
    // ... recordAccessHistoryなど ...
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
    modelTreeList.innerHTML = '';
    parsedWSCenID = "";
    parsedPJNo = "";
    updateInfoPanel();
    // ... 他のリセット処理 ...

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
        
        // 3. ***重要な最適化***
        // 全てのOBJファイルの中身を一つの巨大な文字列に結合します。
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
        const scale = 150 / maxDim;
        loadedObjectModelRoot.scale.set(scale, scale, scale);
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        const allIds = loadedObjectModelRoot.children.map(child => {
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

// --- 他のヘルパー関数（変更なし） ---

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
            let category = elementIdDataMap.has(displayId) ? (elementIdDataMap.get(displayId)['カテゴリー名'] || "カテゴリー無し") : "名称未分類";
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
    // ... (この関数の内容は変更ありません)
}

function createObjectNode(object, parentULElement, depth) {
    // ... (この関数の内容は変更ありません)
}

function frameObject(objectToFrame) {
    // ... (この関数の内容は変更ありません)
}

function handleSelection(target) {
    // ... (この関数の内容は変更ありません)
}

// ... (applyHighlight, removeAllHighlights, zoomToAndIsolate, deIsolateAllObjects, updateInfoPanel, イベントリスナーなど、残りの関数は全て変更ありません)

function animate() {
    requestAnimationFrame(animate);
    controls.update();
    // *** 黒い背景の問題を修正 ***
    // レンダラーに自動で背景をクリアさせないように設定します。
    // これにより、最初に描画したグラデーション背景が保持されます。
    renderer.autoClear = false;
    renderer.render(scene, camera);
}
