 /* Main container for the entire application layout */
        #app-container {
            display: flex;
            flex-direction: column;
            height: 100vh;
        }

        /* Header / Navigation Bar Styles */
        #top-nav {
            flex-shrink: 0;
            height: 50px;
            background-color: #2c3e50;
            color: white;
            display: flex;
            align-items: center;
            justify-content: space-between;
            padding: 0 20px;
            box-sizing: border-box;
            z-index: 50;
        }

        .nav-title {
            font-size: 1.2em;
            font-weight: bold;
            position: absolute;
            left: 50%;
            transform: translateX(-50%);
        }

        .nav-right-group {
            display: flex;
            align-items: center;
            gap: 10px;
        }

        .nav-right-group label {
            font-size: 0.9em;
        }

        .nav-right-group select {
            padding: 5px;
            border-radius: 4px;
            border: 1px solid #7f8c8d;
            background-color: #34495e;
            color: white;
            font-size: 1em;
        }

        /* Container for the two main columns */
        #main-content {
            display: flex;
            flex-grow: 1;
            overflow: hidden;
        }
        
        /* Model Tree Panel (Left Column) */
        #modelTreePanel {
            flex-basis: 320px;
            flex-shrink: 0;
            display: flex;
            flex-direction: column;
            background-color: #2c2c2c;
            color: #e0e0e0;
            overflow: hidden;
        }
        #modelTreePanel .panel-header {
            padding: 10px 15px;
            font-weight: bold;
            border-bottom: 1px solid #444;
            background-color: #383838;
            flex-shrink: 0;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        #modelTreePanel .panel-header input[type="text"] {
            width: calc(100% - 70px);
            box-sizing: border-box;
            padding: 6px 8px;
            background-color: #4f4f4f;
            border: 1px solid #666;
            color: #e0e0e0;
            border-radius: 4px;
            margin-left: 10px;
        }
        #modelTreePanel .close-button {
            cursor: pointer;
            font-size: 20px;
            line-height: 1;
        }
        #modelTreeContainer {
            flex-grow: 1;
            overflow-y: auto;
        }
        #modelTreePanel ul { list-style-type: none; padding: 0; margin: 0; }
        #modelTreePanel .tree-item {
            padding: 6px 10px 6px 0;
            border-bottom: 1px solid #3a3a3a;
            display: flex;
            align-items: center;
            cursor: pointer;
            font-size: 14px;
        }
        #modelTreePanel .tree-item:hover { background-color: #3f3f3f; }
        #modelTreePanel .tree-item.selected { background-color: #007bff; color: white; }
        #modelTreePanel .toggler {
            display: inline-block;
            width: 20px;
            text-align: center;
            cursor: pointer;
            user-select: none;
            margin-right: 5px;
        }
        #modelTreePanel .toggler.empty-toggler { color: transparent; }
        #modelTreePanel .group-name {
            flex-grow: 1;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
            margin-right: 10px;
        }
        #modelTreePanel .visibility-toggle {
            cursor: pointer;
            font-size: 16px;
            margin-left: auto;
            padding: 0 5px;
        }
        #modelTreePanel .visibility-toggle.visible-icon::before { content: '👁️'; }
        #modelTreePanel .visibility-toggle.hidden-icon::before { content: '🚫'; }
        #modelTreePanel ul ul { padding-left: 0; }

        /* Viewer Container (Right Column) */
        #viewer-container {
            flex-grow: 1;
            position: relative;
            overflow: hidden;
            background-color: #cccccc; /* Same as scene background */
        }
        #viewer-container canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        /* Info Panel (Positioned inside viewer-container) */
        #objectInfo {
            position: absolute;
            top: 10px;
            left: 10px;
            background-color: rgba(0,0,0,0.7);
            color: white;
            padding: 10px;
            border-radius: 5px;
            font-family: monospace;
            white-space: pre-wrap;
            z-index: 10;
        }

        /* Loader (Centered over the viewer-container) */
        #loader-container {
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            display: none; /* Hidden by default, shown by JS */
            flex-direction: column;
            justify-content: center;
            align-items: center;
            background-color: rgba(204, 204, 204, 0.7);
            z-index: 1000;
        }
        #loader {
            border: 12px solid #f3f3f3;
            border-radius: 50%;
            border-top: 12px solid #3498db;
            width: 80px;
            height: 80px;
            animation: spin 2s linear infinite;
        }
        #loader-text {
            margin-top: 20px;
            color: #333;
            font-size: 16px;
            font-weight: bold;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body>
    <div id="app-container">
        <!-- Top Navigation Bar -->
        <div id="top-nav">
            <!-- This empty div acts as a spacer for the left side -->
            <div></div> 

            <div class="nav-title">CCC 3D Viewer</div>
            
            <div class="nav-right-group">
                <label for="model-selector">Select Model:</label>
                <select id="model-selector">
                    <option value="GF">GF本社移転</option>
                    <option value="BIKEN">BIKEN (Placeholder)</option>
                    <!-- Add more <option> tags for new models -->
                </select>
            </div>
        </div>
        
        <!-- Main Content Area with two columns -->
        <div id="main-content">
            <!-- Left Column: Model Tree -->
            <div id="modelTreePanel">
                <div class="panel-header">
                    <span>モデル</span>
                    <!-- The close button was removed for this layout, but can be re-added if needed -->
                    <!-- <span class="close-button" id="closeModelTreeBtn" title="Close">×</span> -->
                </div>
                <div id="modelTreeContainer">
                    <ul id="modelTreeList"></ul>
                </div>
            </div>

            <!-- Right Column: 3D Viewer -->
            <div id="viewer-container">
                <!-- The Three.js canvas will be appended here by the script -->
                <div id="loader-container">
                    <div id="loader"></div>
                    <div id="loader-text">Loading 3D Model...</div>
                </div>
                <div id="objectInfo">None</div>
            </div>
        </div>
    </div>
