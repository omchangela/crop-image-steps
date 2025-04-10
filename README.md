Here’s a **step-by-step guide** to implement image preview, cropping, and uploading functionality using **Cropper.js** in your Laravel application. This guide assumes you already have a Laravel project set up and are using Blade templates for the frontend.

---

### Step 1: Install Required Libraries
You need to include **Cropper.js** in your project. You can use a CDN for simplicity.

1. Add the following lines to your Blade template (e.g., `resources/views/admin/instagram/create.blade.php`) inside the `<head>` section:

    ```html
    <!-- Include Cropper.js CSS -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/cropperjs/1.6.1/cropper.min.css" integrity="sha512-hvNR0F/e2J7zPPfLC9auFe3/SE0yG4aJCOd/qxew74NN7eyiSKjr7xJJMu1Jy2wf7FXITpWS1E/RY8yzuXN7VA==" crossorigin="anonymous" referrerpolicy="no-referrer" />

    <!-- Include Cropper.js JS -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/cropperjs/1.6.1/cropper.min.js" integrity="sha512-9KkIqdfN7ipEW6B6k+Aq20PV31bjODg4AA52W+tYtAE0jE0kMx49bjJ3FgvS56wzmyfMUHbQ4Km2b7l9+Y/+Eg==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
    ```

---

### Step 2: Update the Blade Template
Add the HTML structure for the image preview, cropping area, and hidden input for the cropped image.

1. Replace the existing form in your Blade template with the following code:

    ```html
    
            <form action="{{ isset($image) ? route('admin.instagram.update', $image->id) : route('admin.instagram.store') }}" method="POST" enctype="multipart/form-data">
                @csrf
                @if(isset($image))
                    @method('PUT')
                @endif

                <div class="form-group mb-4">
                    <label for="image" class="form-label fw-bold">Instagram Image</label>
                    <input type="file" name="image" id="image" class="form-control" accept="image/*" {{ isset($image) ? '' : 'required' }}>

                    <!-- Image Preview and Cropping Area -->
                    <div class="mt-3">
                        <p class="fw-bold">Preview:</p>
                        <div id="image-preview-container" class="text-center" style="display: none;">
                            <img id="image-preview" src="#" alt="Preview" class="img-fluid rounded shadow-sm" style="max-height: 300px;">
                        </div>
                    </div>

                    @if(isset($image))
                        <div class="mt-3">
                            <p class="fw-bold">Current Image:</p>
                            <img src="{{ asset('storage/' . $image->image) }}" alt="Current Image" class="img-thumbnail" width="150">
                        </div>
                    @endif
                </div>

                <!-- Hidden input to store cropped image data -->
                <input type="hidden" name="cropped_image" id="cropped-image">

                <div class="form-group mt-4">
                    <button type="submit" class="btn btn-primary text-bold">
                        <i class="icon-check"></i> {{ isset($image) ? 'Update Image' : 'Add Image' }}
                    </button>
                </div>
            </form>
    ```

---

### Step 3: Add JavaScript for Cropping Functionality
Add the JavaScript code to handle image preview, cropping, and storing the cropped image data.

1. Add the following script at the bottom of your Blade template (before the closing `</body>` tag):

    ```html
    <script>
    document.addEventListener('DOMContentLoaded', function () {
        const imageInput = document.getElementById('image');
        const imagePreviewContainer = document.getElementById('image-preview-container');
        const imagePreview = document.getElementById('image-preview');
        const croppedImageInput = document.getElementById('cropped-image');
        let cropper;

        imageInput.addEventListener('change', function (e) {
            const files = e.target.files;

            if (files && files.length > 0) {
                const file = files[0];
                const reader = new FileReader();

                reader.onload = function (event) {
                    imagePreviewContainer.style.display = 'block';
                    imagePreview.src = event.target.result;

                    if (cropper) {
                        cropper.destroy();
                    }
                    cropper = new Cropper(imagePreview, {
                        aspectRatio: 9 / 16, // Customize as needed
                        viewMode: 1,
                        autoCropArea: 1,
                        crop: function () {
                            // Automatically update the cropped image data when the crop box changes
                            const croppedImageData = cropper.getCroppedCanvas().toDataURL('image/jpeg');
                            croppedImageInput.value = croppedImageData;
                        }
                    });
                };

                reader.readAsDataURL(file);
            }
        });

        // Clean up cropper on form submit to ensure latest data is sent
        document.querySelector('form').addEventListener('submit', function () {
            if (cropper) {
                const croppedImageData = cropper.getCroppedCanvas().toDataURL('image/jpeg');
                croppedImageInput.value = croppedImageData;
                cropper.destroy();
                cropper = null;
            }
        });
    });
    </script>
    ```

---

### Step 4: Update the Controller
Update your controller to handle the cropped image data and save it to the server.

1. In your controller (e.g., `InstagramController`), update the `store` method:

    ```php
    use Illuminate\Support\Facades\Storage;

    public function store(Request $request)
    {
        $request->validate([
            'image' => 'required|image|mimes:jpeg,png,jpg,gif|max:12048',
            'cropped_image' => 'sometimes|string', // Cropped image as base64
        ]);
    
        if ($request->has('cropped_image') && $request->cropped_image) {
            // Decode the base64 image
            $croppedImage = base64_decode(preg_replace('#^data:image/\w+;base64,#i', '', $request->cropped_image));
    
            // Save the cropped image to storage
            $imageName = 'instagram_' . time() . '.jpg';
            Storage::disk('public')->put('instagram/' . $imageName, $croppedImage);
    
            // Save the image path to the database
            InstagramShowcase::create([
                'image' => 'instagram/' . $imageName,
            ]);
        }
    
        return redirect()->route('admin.instagram.index')->with('success', 'Image uploaded successfully.');
    }
    ```

---

### Step 5: Test the Implementation
1. Open the form in your browser.
2. Select an image using the file input.
3. Crop the image using the cropping tool.
4. Click the "Crop Image" button to finalize the crop.
5. Submit the form.
6. Verify that the cropped image is saved to the `storage/app/public/instagram` directory and the path is stored in the database.

---

### Notes
- Ensure the `storage/app/public` directory is linked by running:
  ```bash
  php artisan storage:link
  ```
- Adjust the `aspectRatio` in Cropper.js to fit your requirements (e.g., square, rectangle, etc.).
- You can customize the image quality and format in the `toDataURL` method (e.g., `image/png` or `image/jpeg`).

This implementation will allow users to preview, crop, and upload images seamlessly.
