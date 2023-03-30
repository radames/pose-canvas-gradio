# Pose Detection Gradio Custom Component

This is a custom Svelte component for [Gradio](https://gradio.app) that uses [mediapipe pose detection](https://google.github.io/mediapipe/solutions/pose_classification.html) to detect pose landmarks in an image. Given a body position, it creates a conditioning image used alongside the input prompt to generate an image. The base model is the [lllyasviel/sd-controlnet-openpose](https://huggingface.co/lllyasviel/sd-controlnet-openpose).

## How to Test  

```bash
npm run dev
```

## How to Build

```bash
npm run build
```

After building your custom component will be in the `dist` folder. The single `index.js` can now be used as a custom component in Gradio read more about how to use on your Gradio app [here](https://github.com/radames/face-canvas-gradio/blob/main/custom_gradio_component.md)

## How to Use in Gradio

Note at the code below, we're using Gradio file server to serve the `index.js` located at the root level of your Gradio app `app.py`. This is done using script source `script.src = "file=index.js"` notation. But you can also use a CDN or any other way to serve the `index.js` file as long as it's served as `content-type: application/javascript`.

```python
import gradio as gr
import requests 
from io import BytesIO
from PIL import Image
import base64

canvas_html = "<pose-canvas id='canvas-root' style='display:flex;max-width: 500px;margin: 0 auto;'></pose-canvas>"
load_js = """
async () => {
  const script = document.createElement('script');
  script.type = "module"
  script.src = "file=index.js"
  document.head.appendChild(script);
}
"""
get_js_image = """
async (canvasData) => {
  const canvasEl = document.getElementById("canvas-root");
  const data = canvasEl? canvasEl._data : null;
  return data
}
"""

def predict(canvas_data):
  base64_img = canvas_data['image']
  image_data = base64.b64decode(base64_img.split(',')[1])
  image = Image.open(BytesIO(image_data))
  return image

blocks = gr.Blocks()
with blocks:
  canvas_data = gr.JSON(value={}, visible=False)
  with gr.Row():
    with gr.Column(visible=True) as box_el:
        canvas = gr.HTML(canvas_html,elem_id="canvas_html")
    with gr.Column(visible=True) as box_el:
        image_out = gr.Image()

  btn = gr.Button("Run")
  btn.click(fn=predict, inputs=[canvas_data], outputs=[image_out], _js=get_js_image)

  blocks.load(None, None, None, _js=load_js)
```
