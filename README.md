# Depth Map Generator

An AI-powered web application that generates depth maps from images using the Depth Anything model via Transformers.js. Built with SvelteKit and deployed on Cloudflare Pages.

## Features

- **AI-Powered Depth Estimation**: Uses the Depth Anything model to generate accurate monocular depth maps
- **Interactive Upload**: Drag-and-drop or click to upload images (PNG, JPG, WEBP)
- **Real-time Adjustments**: Control depth map visualization with interactive sliders:
  - Brightness adjustment (0.1x - 2.0x)
  - Contrast adjustment (0.1x - 3.0x)
  - Color visualization toggle (Blue = near, Red = far)
- **Side-by-side Comparison**: View original image and depth map simultaneously
- **Download Capability**: Save generated depth maps as PNG files
- **Fully Client-side**: All processing happens in the browser - no server required
- **Responsive Design**: Works seamlessly on desktop and mobile devices

## Tech Stack

- **Frontend**: SvelteKit 2.x with Svelte 5 (using runes)
- **AI/ML**: Transformers.js 3.x with Depth Anything model
- **Styling**: Tailwind CSS 4.x
- **Deployment**: Cloudflare Pages/Workers
- **Language**: TypeScript (strict mode)

## Getting Started

### Prerequisites

- Node.js 18+ or higher
- pnpm (recommended) or npm

### Installation

1. Clone the repository:
```bash
git clone <repository-url>
cd depth-map-generator
```

2. Install dependencies:
```bash
pnpm install
```

3. Start the development server:
```bash
pnpm run dev
```

4. Open your browser and navigate to `http://localhost:5173`

## Usage

1. **Upload an Image**:
   - Drag and drop an image onto the upload area, or
   - Click the upload area to select an image from your device

2. **Generate Depth Map**:
   - Click the "Generate Depth Map" button
   - Wait for the model to process (first load will download the model)

3. **Adjust Visualization** (optional):
   - Use the **Brightness** slider to adjust overall depth map brightness
   - Use the **Contrast** slider to enhance depth differences
   - Toggle **Colorize** to switch between colored and grayscale depth maps

4. **Download**:
   - Click "Download Depth Map" to save the processed depth map to your device

## How It Works

This application uses the [Depth Anything](https://github.com/LiheYoung/Depth-Anything) model, which is a foundation model for monocular depth estimation. The model:

1. Analyzes a single 2D image
2. Predicts the relative distance of every pixel from the camera
3. Generates a depth map where:
   - Darker/Blue areas = closer to camera
   - Brighter/Red areas = farther from camera

The entire process runs **client-side** using [Transformers.js](https://huggingface.co/docs/transformers.js), which uses ONNX Runtime to run the AI model directly in your browser via WebAssembly.

## Development

### Project Structure

```
depth-map-generator/
├── src/
│   ├── routes/
│   │   ├── +page.svelte          # Main depth map generator page
│   │   └── +layout.svelte         # Root layout
│   ├── lib/                       # Shared components and utilities
│   ├── app.html                   # HTML template
│   └── app.css                    # Global styles
├── static/                        # Static assets
├── package.json
├── svelte.config.js               # SvelteKit configuration
├── tailwind.config.ts             # Tailwind CSS configuration
└── wrangler.jsonc                 # Cloudflare Workers configuration
```

### Available Scripts

- `pnpm dev` - Start development server
- `pnpm build` - Build for production
- `pnpm preview` - Preview production build locally
- `pnpm test` - Run tests
- `pnpm deploy` - Deploy to Cloudflare Pages

### Building for Production

```bash
pnpm run build
```

The built application will be in `.svelte-kit/output/` ready for deployment to Cloudflare Pages.

## Deployment

This application is configured to deploy to Cloudflare Pages:

```bash
pnpm run deploy
```

Alternatively, connect your repository to Cloudflare Pages for automatic deployments on every push.

## Model Information

- **Model**: Depth Anything Small HF (`Xenova/depth-anything-small-hf`)
- **Type**: Depth Estimation
- **Size**: ~24.8M parameters
- **Format**: ONNX (optimized for web)
- **License**: Apache 2.0

## Performance Considerations

- **First Load**: The model (~21MB) will be downloaded and cached on first use
- **Subsequent Uses**: Model loads from cache for faster processing
- **Processing Time**: Varies based on image size and device capabilities
- **Browser Support**: Modern browsers with WebAssembly support required

## Browser Compatibility

- Chrome/Edge 90+
- Firefox 89+
- Safari 15+
- Mobile browsers (iOS Safari 15+, Chrome Android)

## Use Cases

Depth maps can be used for:
- 3D reconstruction and modeling
- Visual effects and compositing
- Augmented reality applications
- Portrait mode effects
- Scene understanding and analysis
- Accessibility tools

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is open source and available under the [MIT License](LICENSE).

## Acknowledgments

- [Depth Anything](https://github.com/LiheYoung/Depth-Anything) - Foundation model for depth estimation
- [Transformers.js](https://huggingface.co/docs/transformers.js) - Run transformers in the browser
- [Hugging Face](https://huggingface.co/) - Model hosting and tooling
- [SvelteKit](https://kit.svelte.dev/) - Web framework
- [Cloudflare](https://www.cloudflare.com/) - Edge deployment platform

## Support

For issues, questions, or contributions, please visit the [GitHub repository](https://github.com/jasonyingling/depth-map-generator).
