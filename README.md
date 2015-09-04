

# Camera drivers -> HAL -> Framework
Så här ser kedjan ut. Varje videoframe skickas genom kedjan.
Vi kopplar oss in i HAL och där vill vi ta videoframen, skicka in den till vår mjukvara, modifiera framen och sedan byta ut den som HAL vill skicka upp till framework med den vi har modiferat.

Vår mjukvara är beroende av att ta emot videoframen som en GraphicBuffer för att vi ska kunna skapa en OpenGL textur som innehåller videoframen.
Vi måste alltså få över datat i framen till en GraphicBuffer för varje frame som kommer.
I senare HAL versioner så är det möjligt att helt enkelt allokera våra egna Graphicbuffrar säga till HAL när den startar upp allting när kameran startar att be den använda våra buffrar istället för dom som den redan har allokerat. Vi säger helt enkelt, glöm dina egna buffrar och använd dom här du får av oss istället.
Detta gör då att vi slipper göra en kopiering av videoframen eftersom vi får den direkt från kameran som en Graphicbuffer.

```c++
		if(stream == this->captureStream) {
			//FIXME: Memory leak?
			camera3_stream_buffer_t *wrappedStreamBuffer = new camera3_stream_buffer_t();
			wrappedStreamBuffer->acquire_fence = -1;
			wrappedStreamBuffer->release_fence = -1;
			wrappedStreamBuffer->status = CAMERA3_BUFFER_STATUS_OK;
			wrappedStreamBuffer->stream = streamBuffer.stream;
			sp<GraphicBuffer> cameraBuffer = this->getCameraBuffer(stream->width, stream->height, stream->format, stream->usage | GRALLOC_USAGE_HW_VIDEO_ENCODER);
			double_buffer_t buffers;
			buffers.cameraBuffer = cameraBuffer;
			buffers.frameworkBuffer = new camera3_stream_buffer_t();
			memcpy(buffers.frameworkBuffer, &streamBuffer, sizeof(camera3_stream_buffer_t));
			wrappedStreamBuffer->buffer = &cameraBuffer->handle;
			this->saveDoubleBuffer(wrappedStreamBuffer->buffer, buffers);
			wrappedOutputBuffers[i] = *wrappedStreamBuffer;
		}
}
```

I HAL 1.0 fungerar inte detta eftersom kameran och frameworket vägrar att använda någon annan buffer än dom som dom själva har allokerat. Den accepterar helt enkelt inte vi byter ut deras buffrar mot våra. Det är deras buffrar eller inget som gäller.
Därför måste vi då göra en kopiering för att få över videoframen till vår GraphicBuffer så att vi kan skapa en Opengl textur och sedan måste vi göra en kopiering till där vi kopierar framedatat vi har modiferat tillbaka den till buffer som frameworket vill ha.


 
```c++
	sp<GraphicBuffer> inputBuffer = this->getCameraBuffer(width, height, captureFormat,
			GraphicBuffer::USAGE_SW_WRITE_OFTEN | GraphicBuffer::USAGE_SW_READ_OFTEN | GraphicBuffer::USAGE_HW_VIDEO_ENCODER);
	sp<GraphicBuffer> outputBuffer = this->getCameraBuffer(width, height, captureFormat,
			GraphicBuffer::USAGE_SW_WRITE_OFTEN | GraphicBuffer::USAGE_SW_READ_OFTEN | GraphicBuffer::USAGE_HW_VIDEO_ENCODER);
	int fd = bufferHandle->data[0];
	int size = bufferHandle->data[2];
	//ALOGI("Width=%d Height=%d Size=%d", width, height, size);
	int dupfd = dup(fd);
	void* base = mmap(0, size , PROT_READ|PROT_WRITE, MAP_SHARED, dupfd, 0);
	void *inputPtr;
	status_t result = inputBuffer->lock(GraphicBuffer::USAGE_SW_WRITE_OFTEN, &inputPtr);
	memcpy(inputPtr, base, size);
	inputBuffer->unlock();
	this->processVideoCapture(inputBuffer, outputBuffer);
	void *outputPtr;
	outputBuffer->lock(GraphicBuffer::USAGE_SW_READ_OFTEN, &outputPtr);
	memcpy(base, outputPtr, size);
	outputBuffer->unlock();
```


