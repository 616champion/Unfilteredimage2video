import React, { useState, useEffect } from 'react';
import { useDropzone } from 'react-dropzone';
import { createFFmpeg, fetchFile } from '@ffmpeg/ffmpeg';
import { saveAs } from 'file-saver';

const ffmpeg = createFFmpeg({ log: true });

function App() {
  // State variables
  const [images, setImages] = useState([]);
  const [videoBlob, setVideoBlob] = useState(null);
  const [lastGeneratedUrl, setLastGeneratedUrl] = useState('');
  const [ageVerified, setAgeVerified] = useState(false);
  const [showAgeModal, setShowAgeModal] = useState(true);
  const [prompt, setPrompt] = useState('');
  const [styleEffect, setStyleEffect] = useState('none');
  const [videoType, setVideoType] = useState('realistic');
  const [isProcessing, setIsProcessing] = useState(false);
  const [effectOptions, setEffectOptions] = useState({});
  const [progress, setProgress] = useState(0);
  const [aiToolsEnabled, setAiToolsEnabled] = useState(false);
  const [regenerateTrigger, setRegenerateTrigger] = useState(0);

  // Dropzone setup for uploading images
  const { getRootProps, getInputProps } = useDropzone({
    accept: 'image/*',
    onDrop: (acceptedFiles) => {
      setImages((prev) => [...prev, ...acceptedFiles]);
    },
  });

  // Age verification
  const verifyAge = () => {
    setAgeVerified(true);
    setShowAgeModal(false);
  };

  // Generate video function
  const generateVideo = async () => {
    setIsProcessing(true);
    setProgress(0);
    if (!ffmpeg.isLoaded()) await ffmpeg.load();

    // Clear previous output to avoid conflicts
    try {
      ffmpeg.FS('unlink', 'output.mp4');
    } catch (_) {}

    // Write images to ffmpeg FS
    for (let i = 0; i < images.length; i++) {
      ffmpeg.FS('writeFile', `img${i}.jpg`, await fetchFile(images[i]));
    }

    // Prepare input arguments for ffmpeg
    const inputArgs = [];
    const durationPerImage = 30 / images.length; // seconds
    for (let i = 0; i < images.length; i++) {
      inputArgs.push('-loop', '1', '-t', `${durationPerImage}`, '-i', `img${i}.jpg`);
    }

    // Placeholder for effects - expand as needed
    const effectFilter = getEffectFilter();

    // Run ffmpeg
    try {
      await ffmpeg.run(
        ...inputArgs,
        '-filter_complex', effectFilter,
        '-c:v', 'libx264',
        '-t', '30',
        '-pix_fmt', 'yuv420p',
        'output.mp4'
      );
      const data = ffmpeg.FS('readFile', 'output.mp4');
      const url = URL.createObjectURL(new Blob([data.buffer], { type: 'video/mp4' }));
      setVideoBlob(new Blob([data.buffer], { type: 'video/mp4' }));
      setLastGeneratedUrl(url);
    } catch (err) {
      console.error('Error during ffmpeg processing:', err);
    }
    setIsProcessing(false);
  };

  // Effect placeholder: customize as needed
  const getEffectFilter = () => {
    switch (styleEffect) {
      case 'fantasy':
        return 'vignette'; // placeholder filter
      case 'sci-fi':
        return 'lenscorrection'; // placeholder
      case 'horror':
        return 'cinemaScope'; // placeholder
      case 'none':
      default:
        return '';
    }
  };

  // Save video to device
  const saveVideoToDevice = () => {
    if (videoBlob) saveAs(videoBlob, 'generated_video.mp4');
  };

  // Regenerate video
  const regenerateVideo = () => {
    setRegenerateTrigger(prev => prev + 1);
    generateVideo();
  };

  // Effect: rerun generate on trigger update
  useEffect(() => {
    if (images.length > 0 && ageVerified) {
      generateVideo();
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [regenerateTrigger]);

  // UI rendering
	if (showAgeModal) {
    return (
      <div className="modal">
        <div className="modal-content">
          <h2>Are you 18 or older?</h2>
          <button onClick={verifyAge}>Yes, I am 18+</button>
        </div>
      </div>
    );
  }

  return (
    <div className="app-container">
      {/* Header */}
      <h1>Unfiltered Image-to-Video Generator</h1>
      
      {/* Prompt Input */}
      <div className="section">
        <textarea
          placeholder="Enter your prompt (optional, for AI guidance)..."
          value={prompt}
          onChange={(e) => setPrompt(e.target.value)}
        />
      </div>

      {/* Style Options */}
      <div className="section">
        <label>Visual Style:</label>
        <select onChange={(e) => setStyleEffect(e.target.value)} value={styleEffect}>
          <option value="none">None</option>
          <option value="fantasy">Fantasy</option>
          <option value="sci-fi">Sci-Fi</option>
          <option value="horror">Horror</option>
          <option value="realistic">Realistic</option>
          <option value="fictional">Fictional</option>
          <option value="characters">Characters</option>
        </select>
      </div>

      {/* Video Type */}
      <div className="section">
        <label>Video Type:</label>
        <select onChange={(e) => setVideoType(e.target.value)} value={videoType}>
          <option value="realistic">Realistic</option>
          <option value="fictional">Fictional</option>
          <option value="characters">Characters</option>
          <option value="horror">Horror</option>
          <option value="fantasy">Fantasy</option>
        </select>
      </div>

      {/* AI Tools Placeholder */}
      <div className="section">
        <h3>AI-Assisted Creation</h3>
        <button onClick={() => alert('Placeholder for AI tools or API integrations')}>
          Use AI Tools
        </button>
        {/* Expand: integrate APIs for image generation, prompts, etc. */}
      </div>

      {/* Upload images */}
      <div {...getRootProps()} className="dropzone">
        <input {...getInputProps()} />
        <p>Drop unfiltered images here or click to select</p>
      </div>

      {/* Image list */}
      <ul>
        {images.map((file, index) => (
          <li key={index}>{file.name}</li>
        ))}
      </ul>

      {/* Action Buttons */}
      <div className="buttons">
        <button onClick={generateVideo} disabled={isProcessing || images.length === 0}>
          {isProcessing ? 'Processing...' : 'Generate 30s Video'}
        </button>
        {lastGeneratedUrl && (
          <>
            <video src={lastGeneratedUrl} controls width="100%" style={{ marginTop: '10px' }} />
            <div className="download-buttons">
              <button onClick={saveVideoToDevice}>Save to Device</button>
              <button onClick={regenerateVideo}>Regenerate</button>
            </div>
          </>
        )}
      </div>

      {/* Styling for responsiveness and layout */}
      <style jsx>{`
        .app-container {
          max-width: 600px; margin: auto; padding: 10px; font-family: Arial, sans-serif;
        }
        h1 {
          text-align: center; margin-bottom: 20px;
        }
        .section {
          margin-bottom: 15px;
        }
        textarea {
          width: 100%; height: 60px; padding: 5px;
        }
        select {
          width: 100%; padding: 5px;
        }
        .dropzone {
          border: 2px dashed #999; padding: 20px; text-align: center; margin-bottom: 10px; cursor: pointer;
        }
        ul {
          list-style: none; padding-left: 0;
        }
        li {
          margin: 5px 0;
        }
        .buttons {
          display: flex; flex-direction: column; gap: 10px; margin-top: 20px;
        }
        .download-buttons {
          display: flex; gap: 10px; margin-top: 10px;
        }
        button {
          padding: 10px; cursor: pointer; width: 100%;
        }
        .modal {
          position: fixed; top: 0; left: 0; right: 0; bottom: 0;
          background: rgba(0,0,0,0.8); display: flex; justify-content: center; align-items: center;
        }
        .modal-content {
          background: #fff; padding: 20px; border-radius: 8px; text-align: center;
        }
      `}</style>
    </div>
  );
}

export default App;
