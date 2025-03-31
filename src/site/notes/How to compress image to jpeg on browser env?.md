In some scenarios case of real world application we must optimize image before send to server, so this is a some way we can compress image on browser env:

1. Using Canvas
```js
function compressImageJpeg(imageFile, quality = 0,85) {
	return new Promise((resolve, reject) => {
		const reader = new FileReader();
		reader.readAsDataURL(imageFile);

		reader.onload = function (event) {
			const img = new Image();
			img.src = event.target.result;

			img.onload = function () {
				const canvas = document.createElement('canvas');
				const ctx = canvas.getContext('2d');

				canvas.width = img.width;
				canvas.height = img.height;

				ctx.drawImage(img, 0, 0);

				const compressDataUrl = canvas.toDataURL('image/jpeg', quality)
			}

			fetch(compressDataUrl)
				.then(res => res.blob())
				.then(blob => {
					resolve({
						blob: blob,
						dataUrl: compressDataUrl,
						width: img.width,
						height: img.height
					})
				})
				.catch(err => reject(err))

			img.onerror = function () {
				reject(new Error('Failed to load image'))
			}
		}

		reader.onerror = function () {
			reject(new Error('Failed to read file'))
		}
	})
}
```

2. Optimal canvas solution
We compress images using canvas and web workers to achieve multi-process image compression, resulting in higher performance and optimized main thread execution.

```js
// worker.js
self.onmessage = function (e) {
	const { bitmap, quanlity, useOffsreenCanvas } = e.data;

	const canvas = new OffscreenCanvas(bitmap.with, bitmap.height);
	const ctx = canvas.getContext('2d');

	ctx.drawImage(bitmap, 0, 0);
	bitmap.close(); // Release the bitmap once we're dont with it

	if (useOffscreenCanvas) {
		canvas.convertToBlob({ type: 'image/jpeg', quality; quality });

		const reader = new FileReader()
		reader.readAsDataURL(blob)
		reader.onloadend = function () {
			const dataUrl = reader.result;
			self.postMessage({
				type: 'result',
				blob: blob,
				dataUrl: dataUrl,
				width: canvas.width,
				height: canvas.height
			})
		}
	} else {
		const dataUrl = canvas.toDataURL('image/jpeg', quality)
		fetch(dataUrl)
			.then(res => res.blob())
			.then(blob => {
				self.postMessage({
					type: 'result',
					blob: blob,
					dataUrl: dataUrl,
					width: canvas.width,
					height: canvas.height
				});
			});
	}
}
```

```js
createImageBitmap(img)
	.then(bitmap => {
		worker.postMessage({
			bitmap: bitmap,
			quality: quality,
			useOffscreenCanvas: supportsOffscreenCanvas
		}, [bitmap]); // Transfer the bitmap to avoid copying
	})
	.catch(err => {
		console.error('Failed to create ImageBitmap:', err);
		// Fallback to main thread if createImageBitmap fails
});
```

3. Using wasm mozjpeg