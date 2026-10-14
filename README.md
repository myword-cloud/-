// --- 初始化画布 ---
const canvas = new fabric.Canvas('c', {
    backgroundColor: '#ffffff',
    preserveObjectStacking: true,
});

// 用于存储背景图对象，方便替换
let bgImageObj = null;

// --- 工具函数：文件转 Base64 ---
function fileToBase64(file) {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => resolve(reader.result);
        reader.onerror = (error) => reject(error);
    });
}

// --- 功能 1：上传图片作为贴纸 ---
document.getElementById('add-sticker-btn').addEventListener('click', () => {
    document.getElementById('sticker-input').click();
});

document.getElementById('sticker-input').addEventListener('change', async (e) => {
    const file = e.target.files[0];
    if (!file) return;

    const base64 = await fileToBase64(file);
    
    fabric.Image.fromURL(base64, (img) => {
        // 限制最大尺寸，避免贴纸过大
        const maxSize = 300;
        if (img.width > maxSize || img.height > maxSize) {
            img.scaleToWidth(maxSize);
            // 如果高度还是超，再按高度缩放
            if (img.height * img.scaleY > maxSize) {
                img.scaleToHeight(maxSize);
            }
        }

        // 设置贴纸属性
        img.set({
            left: fabric.util.getRandomInt(100, 600),
            top: fabric.util.getRandomInt(100, 400),
            cornerColor: '#007bff',
            cornerSize: 10,
            transparentCorners: false,
            padding: 5,
        });

        canvas.add(img);
        canvas.setActiveObject(img);
        canvas.requestRenderAll();
    });

    // 清空 input，允许重复上传同一文件
    e.target.value = '';
});

// --- 功能 2：上传图片作为背景 ---
document.getElementById('add-bg-btn').addEventListener('click', () => {
    document.getElementById('bg-input').click();
});

document.getElementById('bg-input').addEventListener('change', async (e) => {
    const file = e.target.files[0];
    if (!file) return;

    const base64 = await fileToBase64(file);

    fabric.Image.fromURL(base64, (img) => {
        // 移除旧背景
        if (bgImageObj) {
            canvas.remove(bgImageObj);
        }

        // 缩放以覆盖画布
        const canvasWidth = canvas.getWidth();
        const canvasHeight = canvas.getHeight();
        const scaleX = canvasWidth / img.width;
        const scaleY = canvasHeight / img.height;
        const scale = Math.max(scaleX, scaleY);

        img.set({
            left: 0,
            top: 0,
            scaleX: scale,
            scaleY: scale,
            selectable: false,   // 不可选中
            evented: false,      // 不响应事件
            hoverCursor: 'default',
        });

        // 背景图应该在最底层
        canvas.add(img);
        canvas.sendToBack(img);
        bgImageObj = img;
        canvas.requestRenderAll();
    });

    e.target.value = '';
});

// --- 功能 3：自由绘画模式 ---
let isDrawingMode = false;
const drawBtn = document.getElementById('draw-btn');

drawBtn.addEventListener('click', () => {
    isDrawingMode = !isDrawingMode;
    canvas.isDrawingMode = isDrawingMode;
    
    if (isDrawingMode) {
        // 配置画笔
        canvas.freeDrawingBrush = new fabric.PencilBrush(canvas);
        canvas.freeDrawingBrush.color = '#333333';
        canvas.freeDrawingBrush.width = 5;
        drawBtn.classList.add('active');
        canvas.defaultCursor = 'crosshair';
    } else {
        drawBtn.classList.remove('active');
        canvas.defaultCursor = 'default';
    }
});

// --- 功能 4：一键描边（像素膨胀方案） ---
document.getElementById('outline-btn').addEventListener('click', () => {
    const obj = canvas.getActiveObject();
    if (!obj || obj.type !== 'image') {
        alert('请先点击选中一张贴纸图片，再使用描边功能。');
        return;
    }

    // 获取原图元素
    const imgEl = obj._element;
    if (!imgEl) {
        alert('无法获取图片元素。');
        return;
    }

    // 调用描边函数（颜色白，宽度根据图片大小自适应，这里设个基础值）
    const outlineWidth = 8;
    const newUrl = addOutline(imgEl, '#ffffff', outlineWidth);

    fabric.Image.fromURL(newUrl, (newImg) => {
        // 继承原贴纸的变换属性
        newImg.set({
            left: obj.left,
            top: obj.top,
            scaleX: obj.scaleX,
            scaleY: obj.scaleY,
            angle: obj.angle,
            originX: obj.originX,
            originY: obj.originY,
            cornerColor: '#007bff',
            cornerSize: 10,
            transparentCorners: false,
            padding: 5,
        });

        canvas.remove(obj);
        canvas.add(newImg);
        canvas.setActiveObject(newImg);
        canvas.requestRenderAll();
    });
});

/**
 * 像素膨胀描边算法
 * @param {HTMLImageElement} img 原图元素
 * @param {string} color 描边颜色
 * @param {number} width 描边宽度
 * @returns {string} 合成后的 base64
 */
function addOutline(img, color, width) {
    const w = img.naturalWidth || img.width;
    const h = img.naturalHeight || img.height;
    const pad = width;

    // 1. 把原图画到临时 canvas
    const src = document.createElement('canvas');
    src.width = w; 
    src.height = h;
    const sctx = src.getContext('2d');
    sctx.drawImage(img, 0, 0);
    const srcData = sctx.getImageData(0, 0, w, h);

    // 2. 准备输出 canvas（比原图大一圈）
    const out = document.createElement('canvas');
    out.width = w + pad * 2;
    out.height = h + pad * 2;
    const octx = out.getContext('2d');

    // 3. 生成描边蒙版：对不透明像素做膨胀
    const mask = octx.createImageData(out.width, out.height);
    const md = mask.data;

    for (let y = 0; y < h; y++) {
        for (let x = 0; x < w; x++) {
            const alpha = srcData.data[(y * w + x) * 4 + 3];
            if (alpha === 0) continue;

            // 以当前像素为中心向外扩散
            for (let dy = -width; dy <= width; dy++) {
                for (let dx = -width; dx <= width; dx++) {
                    // 圆形膨胀，边缘更自然
                    if (dx * dx + dy * dy > width * width) continue;
                    
                    const nx = x + pad + dx;
                    const ny = y + pad + dy;
                    const idx = (ny * out.width + nx) * 4;
                    
                    // 设置蒙版为不透明白色
                    md[idx] = 255;
                    md[idx + 1] = 255;
                    md[idx + 2] = 255;
                    md[idx + 3] = 255;
                }
            }
        }
    }

    // 4. 把蒙版涂成描边色，再叠上原图
    octx.putImageData(mask, 0, 0);
    octx.globalCompositeOperation = 'source-in';
    octx.fillStyle = color;
    octx.fillRect(0, 0, out.width, out.height);

    octx.globalCompositeOperation = 'source-over';
    octx.drawImage(img, pad, pad);

    return out.toDataURL('image/png');
}

// --- 功能 5：删除选中对象 ---
document.getElementById('delete-btn').addEventListener('click', () => {
    const activeObjects = canvas.getActiveObjects();
    if (activeObjects.length) {
        // 如果是背景图，要特殊处理（不直接删，而是取消引用）
        activeObjects.forEach(obj => {
            if (obj === bgImageObj) {
                bgImageObj = null;
            }
            canvas.remove(obj);
        });
        canvas.discardActiveObject();
        canvas.requestRenderAll();
    }
});
