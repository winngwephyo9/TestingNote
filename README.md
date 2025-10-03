 //
                //  **【重要】typeIdを'g'タグから抽出するように修正**
                //
                // 1. OBJファイルの中身から 'g' で始まる行を探す
                const gTagMatch = pair.obj.content.match(/^g\s+(.*)$/m);
                const objectName = gTagMatch ? gTagMatch[1].trim() : '';

                // 2. 'g' タグのオブジェクト名から、最後のアンダースコア以降の数字をtypeIdとして抽出
                const typeIdMatch = objectName.match(/_(\d+)$/);
                const typeId = typeIdMatch ? typeIdMatch[1] : null;

                // 3. ジオメトリIDの抽出はこれまで通り
                const geoIdMatch = pair.obj.content.match(/#\s*GeometryObject\.Id\s*:\s*(\d+)/);
                const insGeoObjX = geoIdMatch ? geoIdMatch[1] : null;
