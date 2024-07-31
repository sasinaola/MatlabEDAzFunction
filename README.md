# MatlabEDAzFunction

A structured approach to deploy MATLAB-based encryption and decryption on Azure using Azure Functions. Here's a comprehensive step-by-step guide:

### 1. Set Up Azure Function App
#### Step 1: Create an Azure Function App
1. **Azure Portal**: Navigate to the Azure Portal and create a new Function App.
2. **Azure CLI**: Use the Azure CLI command to create a Function App:
   ```sh
   az functionapp create --resource-group <ResourceGroupName> --consumption-plan-location <Location> --runtime custom --functions-version 4 --name <FunctionAppName> --storage-account <StorageAccountName>
   ```
3. **Choose a Runtime Stack**: Select "Custom" since MATLAB is not natively supported.

### 2. Prepare MATLAB Code
#### Step 2: Write MATLAB Scripts for Encryption and Decryption
Develop your MATLAB functions to handle the image encryption and decryption logic.

Example MATLAB Script (`encrypt.m`):
```matlab
function encryptedImage = encrypt(inputImagePath, outputImagePath)
    % Load image
    img = imread(inputImagePath);
    
    % Example encryption logic (simple pixel manipulation)
    encryptedImage = bitxor(img, 255);
    
    % Save encrypted image
    imwrite(encryptedImage, outputImagePath);
end
```

Example MATLAB Script (`decrypt.m`):
```matlab
function decryptedImage = decrypt(inputImagePath, outputImagePath)
    % Load encrypted image
    encryptedImage = imread(inputImagePath);
    
    % Example decryption logic (reversing the encryption)
    decryptedImage = bitxor(encryptedImage, 255);
    
    % Save decrypted image
    imwrite(decryptedImage, outputImagePath);
end
```

### 3. Containerize MATLAB Code
#### Step 3: Create a Dockerfile
Create a Dockerfile to package your MATLAB scripts along with the necessary runtime.

Example Dockerfile:
```dockerfile
# Use an Ubuntu base image
FROM ubuntu:20.04

# Install dependencies
RUN apt-get update && apt-get install -y wget unzip libx11-6

# Install MATLAB Runtime
RUN wget -qO- https://ssd.mathworks.com/supportfiles/downloads/R2021b/Release/9/deployment_files/installer/complete/glnxa64/MATLAB_Runtime_R2021b_Update_9_glnxa64.zip \
    && unzip MATLAB_Runtime_R2021b_Update_9_glnxa64.zip -d /tmp/matlab \
    && /tmp/matlab/install -mode silent -agreeToLicense yes

# Set environment variables for MATLAB
ENV LD_LIBRARY_PATH /usr/local/MATLAB/MATLAB_Runtime/v911/runtime/glnxa64:/usr/local/MATLAB/MATLAB_Runtime/v911/bin/glnxa64:/usr/local/MATLAB/MATLAB_Runtime/v911/sys/os/glnxa64
ENV XAPPLRESDIR /usr/local/MATLAB/MATLAB_Runtime/v911/X11/app-defaults

# Copy MATLAB scripts
COPY encrypt.m /usr/local/matlab/
COPY decrypt.m /usr/local/matlab/

# Define the entry point for the container
CMD ["matlab", "-nodisplay", "-nosplash", "-r", "encrypt('inputImage.png', 'encryptedImage.png'); exit;"]
```

#### Step 4: Build and Test Docker Image Locally
```sh
docker build -t matlab-encryption .
docker run -v /local/images:/usr/local/matlab matlab-encryption
```

### 4. Deploy Docker Container to Azure Functions
#### Step 5: Push Docker Image to Azure Container Registry
1. **Create an Azure Container Registry (ACR)** if you haven't already:
   ```sh
   az acr create --resource-group <ResourceGroupName> --name <ACRName> --sku Basic
   ```
2. **Push the Docker image to ACR**:
   ```sh
   az acr login --name <ACRName>
   docker tag matlab-encryption <ACRName>.azurecr.io/matlab-encryption:v1
   docker push <ACRName>.azurecr.io/matlab-encryption:v1
   ```

#### Step 6: Configure the Function App
1. **Set the Docker Image**: In the Azure Portal, set your Function App to use the custom Docker image from ACR.
2. **Environment Variables**: Set necessary environment variables like `LD_LIBRARY_PATH` and `XAPPLRESDIR` for MATLAB Runtime.

### 5. Implement the Azure Function
#### Step 7: Create and Configure Azure Function
1. **Create a Function**: Set up an HTTP-triggered function.
2. **Function Code**: In the function's code, you can call the MATLAB scripts using the appropriate command.
   Example (Python):
   ```python
   import os
   import subprocess
   from azure.functions import HttpRequest, HttpResponse

   def main(req: HttpRequest) -> HttpResponse:
       input_image_path = req.params.get('inputImagePath')
       output_image_path = req.params.get('outputImagePath')

       result = subprocess.run(['matlab', '-nodisplay', '-nosplash', '-r', f"encrypt('{input_image_path}', '{output_image_path}'); exit;"], stdout=subprocess.PIPE)

       return HttpResponse(f"Processed image saved at {output_image_path}", status_code=200)
   ```

### 6. Testing and Validation
#### Step 8: Deploy and Test the Function
1. **Deploy**: Deploy your Function App with the Docker image.
2. **Test**: Test the function using HTTP requests with appropriate parameters.

#### Step 9: Error Handling and Logging
1. **Error Handling**: Implement error handling in your function code.
2. **Logging**: Use Azure Application Insights or other logging mechanisms to monitor the function's performance and troubleshoot any issues.

### Additional Considerations
1. **MATLAB Licensing**: Ensure you have appropriate licenses for MATLAB usage.
2. **Security**: Implement necessary security measures to protect your data and functions.
3. **Performance Optimization**: Optimize your MATLAB code and Azure Function configuration for performance, especially for large images or high request volumes.

By following these steps, you can successfully deploy a MATLAB-based image encryption and decryption solution using Azure Functions. This approach leverages Azure's serverless capabilities while incorporating MATLAB's computational strengths.
