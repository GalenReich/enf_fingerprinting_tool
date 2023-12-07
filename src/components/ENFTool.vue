<template>
  <v-container class="fill-height">
    <v-responsive class="align-center text-center overflow-visible">
      <v-img height="100" src="@/assets/logo.svg" />
      <h1 class="py-2 font-weight-bold">Electrical Grid Fingerprinting Tool</h1>
      <v-card
        title="⚠️ This tool is under construction ⚠️"
        text="Information on this page may be incorrect and this tool should not be used as part of any research or investigation."
        color="amber-lighten-4"
      />
      <!-- Title and Information -->
      <v-row justify="center" class="text-left py-2">
        <v-col cols="12" md="8" lg="6" xl="4">
          <p class="py-2">
            This is a prototype tool for Electrical Network Frequency analysis - a technique for chronolocating audio recordings. 
          </p>
          <p class="py-2">
            Electrical networks, like the UK Grid, create a low pitch hum that may be picked up in the background of audio recordings. By matching the fingerprint of this hum to data from the UK Grid, we can pinpoint when the audio may have been recorded.
          </p>
          </v-col>
      </v-row>

      <!-- Step 1: Load Audio -->
      <v-row justify="center" class="text-left py-2">
        <v-col cols="12" md="8" lg="6" xl="4">
          <h2 class="py-2">
            Step 1: Choose an Audio Recording
          </h2>
          <v-file-input
            class ="mt-2"  
            clearable
            accept="audio/*"
            prepend-icon="mdi-waveform"
            label="Select a sound file to fingerprint..."
            v-model="audioFileWrapper"
            @change="resetTool"
            v-on:click:clear="audioFileWrapper=undefined"
          />
          <v-btn
            color="primary"
            href=""
            rel="noopener noreferrer"
            variant="flat"
            :disabled="audioFileWrapper===undefined"
            @click="processAudio" 
          >
            <v-icon start icon="mdi-cogs"/> {{buttonMessage1}}
          </v-btn>
        </v-col>
      </v-row>

      <!-- Step 2: Verify Fingerprint -->
      <div ref="section2">
      <v-row v-if="showSection2" justify="center" class="text-left py-2" >
        <v-col cols="12" md="8" lg="6" xl="4">
          <h2 class="py-2">
            Step 2: Verify Fingerprint
          </h2>
          <p class="py-2">
            Look at the figure below, and check that the Audio Fingerprint (the white line) matches up with the signal behind it (the brightest yellow and green patches).
          </p>
          <v-img
            cover
            :src="stft_image_url"
          />
          <v-btn
            color="primary"
            href=""
            rel="noopener noreferrer"
            variant="flat"
            :disabled="stft_image_url===''"
            @click="confirmFingerprint"
          >
            <v-icon start icon="mdi-cogs"/> {{buttonMessage2}}
          </v-btn>
        </v-col>
      </v-row>
     </div>

      <!-- Step 3: Select Data Range -->
      <div ref="section3">
      <v-row v-if="showSection3&&grid_data!==undefined" justify="center" class="text-left py-2">
        <v-col cols="12" md="8" lg="6" xl="4">
          <h2 class="py-2">
            Step 3: Select Dates To Search
          </h2>
          <p class="py-2">
            In the next step, the tool will attempt to match the extracted fingerprint to patterns in the UK electrical grid. Currently, the tool only supports searching one month at a time.
          </p>
          <p class="py-2">
            Please choose the month you wish to search.
          </p>
          <v-chip-group mandatory v-model=selected_grid_data_url>
            <v-chip
            v-for="resource in grid_data.result.resources!"
            :value="resource.url">
            {{resource.name.slice(0,-26)}}
            </v-chip>
          </v-chip-group>
          <v-btn
            color="primary"
            href=""
            rel="noopener noreferrer"
            variant="flat"
            :disabled="selected_grid_data_url===undefined"
            @click="correlateFingerprint"
          >
            <v-icon start icon="mdi-cogs"/> {{buttonMessage3}}
          </v-btn>
        </v-col>
      </v-row>
      </div>
      <!-- Step 4: Results -->
      <div ref="section4">
      <v-row v-if="showSection4" justify="center" class="text-left py-2">
        <v-col cols="12" md="8" lg="6" xl="4">
          <h2 class="py-2">
            Step 4: Results
          </h2>
          <v-img
            cover
            :src="xcorr_image_url"
          />
          <v-img
            cover
            :src="match_image_url"
          />
        </v-col>
      </v-row>
      </div>
    </v-responsive>
  </v-container>
</template>

<script setup lang="ts">
  import {FFmpeg} from "@ffmpeg/ffmpeg";
  import { fetchFile, toBlobURL } from '@ffmpeg/util'
  import type { LogEvent } from '@ffmpeg/ffmpeg/dist/esm/types'
  import { ref, nextTick } from 'vue'
  import type { Ref } from 'vue'
  import { loadPyodide } from "pyodide";
  import type {PyodideInterface} from "pyodide"

  // Define the expected response from the UK National Grid API
  interface NationalGridResponse extends JSON{
    help: URL
    success: string;
    result:{
      description: string
      resources: {
        [index: number]: { 
          title: string
          url: URL
          name: string
        }
      }
    }
  }

  // Reactive Vue variables
  // Section references
  const section2:Ref<null|HTMLDivElement> = ref(null)
  const section3:Ref<null|HTMLDivElement> = ref(null)
  const section4:Ref<null|HTMLDivElement> = ref(null)
  // Button messages initial state
  const buttonMessage1:Ref<string> = ref('Process file')
  const buttonMessage2:Ref<string> = ref('Confirm Fingerprint')
  const buttonMessage3:Ref<string> = ref('Begin Matching')
  // Booleans for showing sections
  const showSection2:Ref<boolean> = ref(false)
  const showSection3:Ref<boolean> = ref(false)
  const showSection4:Ref<boolean> = ref(false)
  // URL strings for generated images
  const stft_image_url:Ref<string> = ref('')
  const xcorr_image_url:Ref<string> = ref('')
  const match_image_url:Ref<string> = ref('')
  // More complex data wrappers
  const audioFileWrapper:Ref<undefined|File[]> = ref(undefined)
  const grid_data:Ref<undefined|NationalGridResponse> = ref(undefined)
  const selected_grid_data_url:Ref<undefined|URL> = ref(undefined)

  // Fetch available data from the UK National Grid
  const grid_data_endpoint:string = "https://api.nationalgrideso.com/api/3/action/package_show?id=system-frequency-data"
  fetch(grid_data_endpoint).then((response)=> response.json()).then((response:NationalGridResponse) => grid_data.value = response)

  async function resetTool():Promise<void>{
    // Reset section display
    showSection2.value=false
    showSection3.value=false
    showSection4.value=false

    buttonMessage1.value='Process File'
    buttonMessage2.value='Confirm Fingerprint'
    buttonMessage3.value='Begin Matching'
  }

  async function initializeFFMPEG():Promise<FFmpeg>{
    // Set-up FFMPEG and logging
    const ffmpegBaseURL = 'https://unpkg.com/@ffmpeg/core@0.12.4/dist/esm'
    const ffmpeg = new FFmpeg();
    
    ffmpeg.on('log', ({ message: msg }: LogEvent) => {console.log(msg)})
    await ffmpeg.load({
      coreURL: await toBlobURL(`${ffmpegBaseURL}/ffmpeg-core.js`, 'text/javascript'),
      wasmURL: await toBlobURL(`${ffmpegBaseURL}/ffmpeg-core.wasm`, 'application/wasm'),
    })
    return ffmpeg
  }

  async function initializePyodide():Promise<PyodideInterface>{
    const pyodideIndexURL = "https://cdn.jsdelivr.net/pyodide/v0.24.1/full/"
    const pyodide = await loadPyodide({indexURL: pyodideIndexURL});
    await pyodide.loadPackage(["numpy","pandas","scipy", "matplotlib"])
    await pyodide.runPythonAsync(`
      from scipy import signal, io
      import numpy as np
      import pandas as pd
      from matplotlib import pyplot as plt
    `)
    return pyodide
  }

  const ffmpegPromise = initializeFFMPEG()
  const pyodidePromise = initializePyodide()

  async function processAudio():Promise<void> {
    await resetTool()
    // Wait for FFMPEG to load
    buttonMessage1.value = 'Loading FFMPEG'
    const ffmpeg = await ffmpegPromise

    // Process audio with FFMPEG and save to output.wav
    buttonMessage1.value = 'Processing Audio'
    if(audioFileWrapper.value === undefined) throw new Error("audioFileWrapper is undefined in processAudio()")
    const audioFile = audioFileWrapper.value['0']
    
    await ffmpeg.writeFile(audioFile['name'], await fetchFile(audioFile)) // Write form file to local storage
    await ffmpeg.exec([
      '-loglevel', 'error', // Log level
      '-i', audioFile['name'], // Input file
      '-vn', // Ignore any video
      '-acodec', 'pcm_s32le', // The codec: PCM signed 32-bit little-endian
      '-ar', '300', // Sampling frequency
      '-ac', '2', // Audio channels - we only need 1 really
      '-af', 'bandpass=frequency=100:width_type=h:width=2', // Apply a bandpass filter centered at 100Hz with a 2Hz width
      'output.wav'
    ])

    // Extract fingerprint from processed audio with Pyodide
    buttonMessage1.value = 'Extracting fingerprint'
    const outputData = await ffmpeg.readFile('output.wav')
    var pyodide = await pyodidePromise
    pyodide.FS.writeFile("output.wav", (outputData as Uint8Array), { type: 'audio/wav' });

    await pyodide.runPythonAsync(`
        def generateFingerprint():
          fs, test_signal = io.wavfile.read("output.wav") 

          test_signal=test_signal[:fs*60*60*3,0]

          # Set parameters for the STFT

          nperseg = 10*fs # number of signal samples to include in each FFT (increases the STFT resolution in frequency and decreases it in time)
          nfft = 1024*8 # Number of bins in the fft (this zero pads the signal and is a substitute for quadratic interpolation)

          grid_temporal_resolution=1
          noverlap=nperseg-grid_temporal_resolution*fs # This ensures the temporal resolution of the STFT matches the reference data

          freqs,times,stft = signal.stft(test_signal, fs=fs, nperseg=nperseg, noverlap=noverlap, nfft=nfft) # Apply STFT
          peak_idxs = [idx for t in range(len(times)) if (idx := np.argmax(np.abs(stft[:,t])))] # Extract peak for each point in time
          peak_idxs = signal.medfilt(peak_idxs, kernel_size=29) # Apply a median filter, a larger kernel size means more smoothing
          signal_freqs = freqs[peak_idxs]
          
          # Plot the STFT over the frequency range of interest
          filter_center_frequency = 100
          half_bandwidth = 1
          critical_frequencies = [filter_center_frequency - half_bandwidth, filter_center_frequency + half_bandwidth] 

          freqs_to_show = np.all([freqs>critical_frequencies[0],freqs<critical_frequencies[1]], axis=0)
          mag = stft[peak_idxs, range(len(peak_idxs))]
          nstd = 2
          vmax = nstd*np.nanstd(mag)

          fig = plt.figure(figsize=(10,5))
          plt.pcolormesh(times, freqs[freqs_to_show], np.abs(stft[freqs_to_show,:]),vmin=0, vmax=vmax, shading='gouraud')
          plt.plot(times, signal_freqs,color="white",alpha=0.75)
          plt.title('STFT Magnitude')
          plt.ylabel('Frequency [Hz]')
          plt.xlabel('Time [sec]')
          plt.savefig('STFT.jpg', format='jpg')

          # Manual garbage collection
          del stft, test_signal, fig,
          del peak_idxs, nperseg, nfft, grid_temporal_resolution, noverlap, freqs, 
          del filter_center_frequency, half_bandwidth, critical_frequencies, freqs_to_show, mag, nstd, vmax

          return times, signal_freqs, fs

        times, signal_freqs, fs = generateFingerprint()
        `
        );
    
    // Make STFT image available to Vue
    const image = pyodide.FS.readFile("STFT.jpg");
    stft_image_url.value = URL.createObjectURL(new Blob([image.buffer], {type: "image/jpg"}))

    buttonMessage1.value = 'Finished'
    showSection2.value=true
    await nextTick()
    const div = section2.value!
    div.scrollIntoView({block: "start", inline: "nearest", behavior: "smooth"});    
  }    

  async function confirmFingerprint():Promise<void> {
    showSection3.value=true
    buttonMessage2.value="Fingerprint Confirmed"
    await nextTick()
    const div = section3.value!
    div.scrollIntoView({block: "start", inline: "nearest", behavior: "smooth"});  
  }
  
  async function correlateFingerprint():Promise<void>{
    
    // Load the data from the UK National Grid
    buttonMessage3.value="Loading Grid Data"
    const gridData = await fetch(selected_grid_data_url.value!)
    var pyodide = await pyodidePromise
    pyodide.FS.writeFile("gridData.csv", await gridData.text(), { type: 'type/csv' });

    // Correlate the uploaded audio with the selected grid data
    buttonMessage3.value="Matching Fingerprint"
    await pyodide.runPythonAsync(`
        def correlateFingerprint():
          grid_table = pd.read_csv("gridData.csv")
          
          grid_table['dtm'] = pd.to_datetime(grid_table['dtm'])
          grid_times = grid_table['dtm'].to_numpy()
          grid_freqs = grid_table['f'].to_numpy()

          recording_duration = int(len(times))

          # Cross-correlate test signal peak frequencies with reference frequency data (and normalize)
          
          def moving_average(x, w):
              return np.convolve(x, np.ones(w), 'same') / w

          # When correlating, center the frequencies around zero to avoid very large numbers
          xcorr = signal.correlate(
                  grid_freqs-np.mean(grid_freqs), 
                  signal_freqs-np.mean(signal_freqs),
                  mode='same'
              ) 

          ref_normalization = pd.Series(grid_freqs).rolling(recording_duration, center=True).std()
          signal_normalization = np.std(signal_freqs)

          # Normalize to avoid the absolute magnitude of the grid frequency having an effect if different from the nominal value
          xcorr_norm = xcorr/ref_normalization/signal_normalization/recording_duration

          # Plot the Normalized Cross Correlation
          fig1 = plt.figure(figsize=(10,5))
          plt.plot(grid_times, xcorr_norm)
          plt.xlabel("Time")
          plt.ylabel("Normalized Cross-Correlation (arbitrary units)")
          plt.title("Agreement between test and reference signals")
          plt.savefig('XCorr.jpg', format='jpg')

          # Create a figure overlaying the test and reference signal for the most probable match
          match_center = np.argmax(xcorr_norm)

          match_start = match_center-recording_duration//2
          match_end = match_start+len(signal_freqs)
          plot_idxs = range(match_start, match_end)

          fig2 = plt.figure(figsize=(10,5))
          plt.plot(grid_times[plot_idxs],signal_freqs, alpha=0.8)
          plt.plot(grid_times[plot_idxs], grid_freqs[plot_idxs]*2, alpha=0.8)
          plt.ylabel('Peak Frequency [Hz]')
          plt.xlabel('Time [sec]')
          plt.legend(['Recording Under Test', 'UK Grid'])
          plt.title(f'Possible match beginning at {grid_times[plot_idxs][0]}')
          plt.grid()
          plt.savefig('Match.jpg', format='jpg')

          del grid_table, grid_times, grid_freqs, recording_duration, xcorr, ref_normalization
          del signal_normalization, xcorr_norm, match_center, match_start, match_end, plot_idxs
          del fig1, fig2

        correlateFingerprint()
        `
    );

    // Make images available to Vue
    const image_xcorr = pyodide.FS.readFile("XCorr.jpg");
    xcorr_image_url.value = URL.createObjectURL(new Blob([image_xcorr.buffer], {type: "image/jpg"}))

    const image_match = pyodide.FS.readFile("Match.jpg");
    match_image_url.value = URL.createObjectURL(new Blob([image_match.buffer], {type: "image/jpg"}))

    // Show results
    buttonMessage3.value="Matching Complete"
    showSection4.value = true
    await nextTick()
    const div = section4.value!
    div.scrollIntoView({block: "start", inline: "nearest", behavior: "smooth"});  
  }
</script>
