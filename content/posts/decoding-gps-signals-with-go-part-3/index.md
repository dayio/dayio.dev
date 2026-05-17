+++
date = '2026-05-17T11:15:03+02:00'
title = "Decoding GPS signal with GO (part 3)"
description = "We drop the brute-force for FFT and goroutines, build tracking loops to get a lock, and finally extract the 0x8B GPS preamble from raw noise."
tags = ["gps", "golang", "sdr", "rtl-sdr", "dsp"]
+++

In the last article, we saw how to catch the satellite’s signal using our gold codes and some trickery to handle their movements in the sky using Doppler compensation and phase shifting. However, this comes at a cost. It's slow as hell. And we need to improve this if we want to decode our packets fast.

## The Need for Speed (Acquisition Optimization)

First, let's benchmark the first version:

```go
for {    
    now := time.Now()  
    ...  
  
    for prn := 1; prn <= 32; prn++ {  
       bestPhase, bestDoppler, snr := gps.Acquire(outputComplex, prn, sampleRate)  
		...
    }  
    
    log.Printf("Processing took %v\n", time.Since(now))  
}
```

```bash
$ make run
go build -o bin/gps-decoder cmd/gps-decoder/main.go
./bin/gps-decoder
Satellite PRN 05 found ! Phase:  792 | Doppler: -5000.00 | SNR:  8.71
Satellite PRN 14 found ! Phase:  636 | Doppler: -2000.00 | SNR: 12.41
Satellite PRN 15 found ! Phase:  587 | Doppler: 4000.00 | SNR: 11.78
Satellite PRN 20 found ! Phase:  906 | Doppler:  0.00 | SNR: 16.16
Satellite PRN 22 found ! Phase: 1140 | Doppler:  0.00 | SNR: 13.52
Satellite PRN 24 found ! Phase: 1168 | Doppler: 5000.00 | SNR:  8.11
Processing took 52.352300506s
```

### Goroutines to the rescue

Alright, 52 seconds on an AMD Ryzen 7 PRO 6850U with 8 cores and 16 threads is slow. So now we will run the acquire method concurrently. To do so, we will define a struct for our acquisition data and then distribute them on a channel for later use. This way, we will use all of our cores.

```go
type AcquireResult struct {  
    PRN         int  
    BestPhase   int  
    BestDoppler float64  
    SNR         float64  
}
```

Here I've redefined the PRN because in the channel we lose track of the parameters passed, as we will not be in a synchronous for-loop anymore. Below is the updated method

```go
func Acquire(samples []complex128, prn int, sampleRate float64) AcquireResult {  
    ...  
    return AcquireResult{  
       PRN:         prn,  
       BestPhase:   bestPhase,  
       BestDoppler: bestDoppler,  
       SNR:         snr,  
    }  
}
```

Now, the interesting part is the main function. I've added some goroutines to run the function concurrently and merge the results:

```go
for {  
    now := time.Now()
    ...
  
    var wg sync.WaitGroup  
  
   // Buffering to 32 ensures our 32 PRN goroutines never block when writing their result
    acquireCh := make(chan gps.AcquireResult, 32)  
  
    for prn := 1; prn <= 32; prn++ {  
       wg.Add(1)  
  
       go func(prn int) {  
          defer wg.Done()  
          acquireCh <- gps.Acquire(outputComplex, prn, sampleRate)  
       }(prn)  
    }  
  
    go func() {  
       wg.Wait()  
       close(acquireCh)  
    }()  
  
    for acquireResult := range acquireCh {  
       if acquireResult.SNR > 8.0 {  
          fmt.Printf("Satellite PRN %02d found ! Phase: %4d | Doppler: %5.2f | SNR: %5.2f\n",  
             acquireResult.PRN,  
             acquireResult.BestPhase,  
             acquireResult.BestDoppler,  
             acquireResult.SNR,  
          )  
       }  
    }
    
	log.Printf("Processing took %v\n", time.Since(now))  
}
```

```bash
$ make run 
go build -o bin/gps-decoder cmd/gps-decoder/main.go
./bin/gps-decoder
Satellite PRN 24 found ! Phase: 1168 | Doppler: 5000.00 | SNR:  8.11
Satellite PRN 22 found ! Phase: 1140 | Doppler:  0.00 | SNR: 13.52
Satellite PRN 15 found ! Phase:  587 | Doppler: 4000.00 | SNR: 11.78
Satellite PRN 14 found ! Phase:  636 | Doppler: -2000.00 | SNR: 12.41
Satellite PRN 20 found ! Phase:  906 | Doppler:  0.00 | SNR: 16.16
Satellite PRN 05 found ! Phase:  792 | Doppler: -5000.00 | SNR:  8.71
Processing took 4.833257081s
```

### The Doppler Wall

Now, 4.8 seconds, that's better, but still it's far from being perfect. As you can see, there is a `5000` and `-5000` Hz Doppler shift, and it's our range so it means we need to increase it from `+5000Hz -5000Hz` to `+10 000Hz -10 000Hz` and of course that will slow down our program.

```go
// Doppler scanning from -10000Hz to +10000Hz with 500Hz increments  
for doppler := -10000.0; doppler <= 10000.0; doppler += 500 {
```

```shell
$ make run
go build -o bin/gps-decoder cmd/gps-decoder/main.go
./bin/gps-decoder
Satellite PRN 22 found ! Phase: 1140 | Doppler:  0.00 | SNR: 13.66
Satellite PRN 24 found ! Phase: 1168 | Doppler: 6000.00 | SNR: 10.24
Satellite PRN 05 found ! Phase:  792 | Doppler: -5500.00 | SNR: 10.80
Satellite PRN 21 found ! Phase: 1723 | Doppler: -7500.00 | SNR:  8.34
Satellite PRN 30 found ! Phase:  241 | Doppler: -5500.00 | SNR:  8.54
Satellite PRN 20 found ! Phase:  906 | Doppler:  0.00 | SNR: 16.13
Satellite PRN 14 found ! Phase:  636 | Doppler: -2000.00 | SNR: 12.40
Satellite PRN 15 found ! Phase:  587 | Doppler: 4000.00 | SNR: 11.83
Processing took 9.397554936s
```

And as predicted, if you double the range, you double the execution time. So here, to increase the speed of phase and Doppler shifting, we can't just use more cores, we need to use the **Fast Fourier Transform** (**FFT**).

### The Fast Fourier Transform (FFT)

But what is the FFT exactly? It's not just fancy matrix operations. The FFT allows us to shift our perspective from the _Time Domain_ to the _Frequency Domain_.

Until now, we've brute-forced our way through the 1023 possible phases by sliding our Gold Code one step at a time while accounting for the Doppler frequency. It's linear and not efficient at all.

By using the FFT, we can convert both our signal and our Gold Code into frequencies, multiply them together, and transform the result back. The magic? This single operation gives us the correlation scores for *all 1023 possible phases at the exact same time*. We can literally execute this whole scan in under a second.

To understand why the FFT is great, we have to look at the math behind what we are actually doing. Until now, we computed the correlation manually. For a received signal $x$ and our local Gold Code $c$, finding the correlation score $R$ for a phase shift $\tau$ looks like this:

$$R[\tau] = \sum_{n=0}^{N-1} x[n] \cdot c[n - \tau]$$

Where:
- $N$ is the total length of our Gold Code (**1023** chips)
- $x$ is our received signal buffer
- $c$ is our local Gold Code array

Because we have to calculate this for every single phase shift $\tau$ (from 0 to 1022), the computational complexity is $O(N^2)$. It scales terribly.

And this translates in our current code (without the Doppler compensation) to :

```go
var correlation complex128

// our sigma n = 0 <-> N-1
for i := range 1023 {

    // Shift handling (the "-t" of the formula)
    sigIdx := (phase + int(float64(i)*(sampleRate/1023000.0))) % len(samples)

    // x[n] * c[n - t] (multiplication and accumulation in R)
    correlation += samples[sigIdx] * complex(goldCode[i], 0)
}
```

But in *DSP* (Digital Signal Processing), there is a shortcut called the **Circular Correlation Theorem**. It states that the cross-correlation of two signals in the time domain is mathematically equivalent to multiplying them in the frequency domain, provided we take the complex conjugate of the second signal, and it is written as:

$$\mathcal{F}\{x \star c\} = \mathcal{F}\{x\} \cdot \mathcal{F}\{c\}^*$$

Where:
- $\mathcal{F}$ is the Fourier Transform (our FFT)
- $\star$ is the cross-correlation operator
- $*$ is the complex conjugate (inverting the sign of the imaginary part)

If we want to get our correlation scores back in the time domain (to find our exact `bestPhase`), we simply apply the Inverse Fast Fourier Transform (IFFT) to that result:

$$R = \text{IFFT}( \text{FFT}(x) \cdot \text{FFT}(c)^* )$$

This single equation reduces our computational complexity from $O(N^2)$ to $O(N \log N)$. Instead of a nested loop doing over a million iterations per Doppler step, we do three FFT operations and a simple vector multiplication. It's not even comparable to the previous naive method.

But as I'm not a mathematician and don't want to reinvent the wheel as this article is about GPS and not FFT, we will use [`github.com/mjibson/go-dsp/fft`](https://github.com/madelynnblue/go-dsp), a standard and highly efficient DSP library in the Go ecosystem.

```shell
go get github.com/madelynnblue/go-dsp/fft
```

And that gives us something like this in our code (I'll make a long version below)

```go
func ApplyDoppler(samples []complex128, doppler float64, sampleRate float64, padSize int) []complex128 {
	rotated := make([]complex128, padSize)
	
	for i := range samples {
		angle := 2 * math.Pi * doppler * (float64(i) / sampleRate)
		rotated[i] = samples[i] * cmplx.Exp(complex(0, -angle))
	}
	
	return rotated
}

...

// We apply the Doppler shift to the whole signal at once
rotatedSamples := ApplyDoppler(samples, doppler, sampleRate)

// We transform the signal into frequencies (FFT) using the library
signalFFT := fft.FFT(rotatedSamples)

// We multiply it with our Gold Code frequencies (already tranformed)
for i := range signalFFT {
    signalFFT[i] = signalFFT[i] * codeFFT[i]
}

// And we transform back to time domain the correlationScores and it now contains ALL 1023 phases at once.
correlationScores := fft.IFFT(signalFFT)
```

And as you can see there is no `for` loop anymore for the 1023 phases. Now we need to implement this "pseudocode" into our acquire function and we will create another file named `acquireFFT` so we can compare the performance difference between both algorithms later.

This *FFT* algorithm needs a power of two input array sizes in order to run exponentially fast (like 512, 1024, 2048) and our Gold Code is 1023 chips long and our sample buffer is 2000...

The fix? We need to pad both arrays with zeros until they reach a length of 2048. This is a standard DSP technique called **Zero-Padding**. It doesn't change the frequencies, it just gives the algorithm the mathematical space it needs.

```go
func AcquireFFT(samples []complex128, prn int, sampleRate float64) AcquireResult {
	
	goldCode := GenerateGoldCode(prn)  
	  
    // Filling the remaining 48 indices with zeros
	N := 2048  
	paddedCode := make([]complex128, N)  
	  
	for i := 0; i < len(samples); i++ {  
		// stretch the signal to fit the 1023-bit code  
		chipIdx := int(float64(i)*1023000.0/sampleRate) % 1023  
		paddedCode[i] = complex(goldCode[chipIdx], 0)  
	}  
	  
	// Compute FFT of the local Gold Code
	// We only need to do this once per PRN and not inside the Doppler loop !
	codeFFT := fft.FFT(paddedCode)  
	  
    // Take the complex conjugate of the Code FFT
	for i := range codeFFT {  
		codeFFT[i] = cmplx.Conj(codeFFT[i])  
	}	

	maxPower := 0.0
	sumPower := 0.0
	bestPhase := 0
	bestDoppler := 0.0
	
	// Next step is the Doppler Loop and the IFFT as we saw above
}
```

We will add the function `ApplyDoppler` to our file as defined above (so don't worry if you don't see it now, it's there but not there, got it?) but this function will "rotate" the frequency so, [get rotated...](https://knowyourmeme.com/memes/get-rotated-idiot)

The rest of the function is pretty similar as above and will just do almost the same and then find a peak in the correlations :

```go
// Doppler scanning  
for doppler := -10000.0; doppler <= 10000.0; doppler += 500 {  
  
    // Rotation and padding  
    rotatedSamples := ApplyDoppler(samples, doppler, sampleRate, N)  
  
    // Go to frequency domain  
    signalFFT := fft.FFT(rotatedSamples)  
  
    // Circular correlation (Multiplication)  
    for i := range signalFFT {  
       signalFFT[i] = signalFFT[i] * codeFFT[i]  
    }  
  
    // Back to temporal domain  
    correlationScores := fft.IFFT(signalFFT)  
  
    // Peak correlation analysis without padding values  
    for i := 0; i < len(samples); i++ {  
       power := cmplx.Abs(correlationScores[i])  
  
       sumPower += power  
       count++  
  
       if power > maxPower {  
          maxPower = power  
          bestPhase = i  
          bestDoppler = doppler  
       }  
    }  
}  

// Return the results
```

Now the moment of truth

```shell
$ make run
go build -o bin/gps-decoder cmd/gps-decoder/main.go
./bin/gps-decoder
Satellite PRN 15 found ! Phase:  585 | Doppler: 1500.00 | SNR: 11.46
Satellite PRN 22 found ! Phase: 1187 | Doppler:  0.00 | SNR:  6.74
Satellite PRN 21 found ! Phase: 1770 | Doppler: -3500.00 | SNR:  8.10
Satellite PRN 24 found ! Phase: 1214 | Doppler: 3000.00 | SNR:  5.53
Satellite PRN 30 found ! Phase:  239 | Doppler: -2500.00 | SNR:  7.39
Satellite PRN 20 found ! Phase:  904 | Doppler: 500.00 | SNR:  8.62
Satellite PRN 05 found ! Phase:  790 | Doppler: -3000.00 | SNR:  7.33
Satellite PRN 23 found ! Phase: 1681 | Doppler: 3000.00 | SNR:  5.15
Satellite PRN 14 found ! Phase:  634 | Doppler: -1000.00 | SNR:  9.72
Processing took 109.325876ms
```

From almost 10 seconds, down to 110 milliseconds, that's what I call a little improvement!

## From Snapshots to Video Feed (Tracking Loops)

Now that we have our acquisition part done, we need to keep the tracking with all of those satellites because we can compare acquisition to a blurry snapshot at time T, while tracking is like a high-definition video feed of the moving satellite. Remember we still need to extract the data, and if we lose contact or track, we can't decode the packets.

If you recall perfectly from the first article, the data flows at 50 bits per second... using *BPSK* (Binary Phase Shift Keying). So for the tracking we will use two methods :
- The PLL (Phase Locked Loop) is the tracking of the frequency. Because our satellite moves, the Doppler also moves, so we need to track and adjust.
- The DLL (Delay Locked Loop) is the tracking of the phase. The same as above, the gold code should stay aligned, and if it moves, we must delay it or forward it.

For the PLL and DLL I haven't invented it, it is defined [here in the Navipedia](https://gssc.esa.int/navipedia/index.php/Tracking_Loops) and I'll refer to it when we code our PLL and DLL functions and tests.

### The Channel State Machine

Let's code a tracking loop that will maintain the locks, but it will require a rewrite of our main function, and we will make a continuous search on all 32 PRN. In a pseudocode this will resemble as :

```go
func main() {
	
	// Create channels for our 32 satellites
	channels := make([]chan []complex128, 32)
	
	for prn := 1; prn <= 32; prn++ {
		// Define a buffer 
		channels[prn-1] = make(chan []complex128, 10) 
		
		// Start a state machine goroutine for this PRN
		go gps.RunChannel(prn, channels[prn-1], sampleRate)
	}

	// Get the input buffer
	inputBuffer := // Get the data from the SDR

	// The broadcast loop
	for {
		// Convert raw bytes to complex128
		complexChunk := dsp.ToComplex(inputBuffer)

		// Broadcast the same chunk to all 32 listening goroutines
		for i := 0; i < 32; i++ {
			channels[i] <- complexChunk
		}
	}
	
	// Close channels to let goroutines exit cleanly
	for i := 0; i < 32; i++ {
		close(channels[i])
	}
}
```

The `RunChannel` function will be a "state machine" as there will be two distinct modes: *acquisition* and *tracking*. And based on the *SNR* it will switch from one to another mode. Also in the state *tracking* we will run a `TrackChunk` function to process 1ms of data and check if satellite is gone and if not, pass the data to a channel for decoding elsewhere.

```go
// Define an "enum"
type ChannelState int

const (
	StateAcquiring ChannelState = iota
	StateTracking
)

func RunChannel(prn int, sampleStream <-chan []complex128, sampleRate float64) {
	state := StateAcquiring
	
	var trackState TrackingState

	// The goroutine stays alive and waits for 1ms chunks of signal
	for chunk := range sampleStream {

		switch state {

		case StateAcquiring:
			// We accumulate enough chunks internally to do a full FFT (e.g., 2ms)
			// For simplicity in this pseudo-code, assume chunk is big enough
			acqResult := AcquireFFT(chunk, prn, sampleRate)

			if acqResult.SNR > 8.0 {
				fmt.Printf("[PRN %02d] Acquisition done - SNR: %.2f. Switching to TRACKING.\n", prn, acqResult.SNR)
				
				// Initialize tracking
				trackState = TrackingState{
					PRN:          prn,
					CodePhase:    float64(acqResult.BestPhase),
					CarrierFreq:  acqResult.BestDoppler,
					CarrierPhase: 0.0,
				}
				
				state = StateTracking
			}

		case StateTracking:
		
			// Process the 1ms chunk to adjust our loops using PLL and DLL.
			trackResult := TrackChunk(chunk, &trackState, sampleRate)

			// If the prompt correlator energy drops, the satellite is gone
			if trackResult.PromptPower < 50.0 { // Arbitrary value
				fmt.Printf("[PRN %02d] Tracking lost - Fallback to ACQUISITION.\n", prn)
				state = StateAcquiring
				continue
			}

			// If we are locked, we can start buffering the I values (bits)
			if trackResult.IsLocked {
				ProcessNavigationData(prn, trackResult.I_Value)
			}
		}
	}
}
```

And the `TrackChunk` method applies the *PLL* and *DLL* on a single 1ms chunk and updates the state.

### PLL & DLL (We're not on Windows)
#### The global schema (I&D -> Discriminator -> Filter -> NCO)

> "In its most common implementation, the receiver implements code and carrier tracking loops [...] The tracking loops include: Integrate & Dump (I&D), Discriminators, Filters, NCO." - Navipedia

We need to implement the four stages in order to properly keep a lock :
1. *I&D (Integrate & Dump)* : It's our `Correlate` function that will add all the values and dumps the **Early**, **Prompt**, **Late** for the I/Q data and the carrier wave.
2. *Discriminators* : Math formulas that measure the errors (how much we've drifted)
3. *Loop Filters* : We don't react abruptly to errors, we must smooth out the correction to not increase the noise.
4. *NCO (Numerical Control Oscillator)* : It's the "fixer" it takes the filtered value, adjusts the Doppler, and gives it to the next chunk (millisecond).

#### DLL (Delay Lock Loop): Follow the code

> "While code tracking loops follow the code delay of the incoming signal using Delay Lock Loops (DLL)" - Navipedia


The DLL ensures that our local code moves at exactly the same speed as the code received from the satellite. Its discriminator compares the signal strength of the “early” and “late” signals. The standard discriminator is the [**Normalized Early minus Late Envelope**](https://gssc.esa.int/navipedia/index.php?title=Delay_Lock_Loop_(DLL)#:~:text=Normalized%20Early%20minus%20Late%20Envelope%2C%20given%20by%3A)
$$E = \frac{\text{Early} - \text{Late}}{\text{Early} + \text{Late}}$$

#### PLL (Phase Lock Loop): Follow the carrier wave

> "Carrier tracking loops can be designed to follow either the phase of the incoming signal – using Phase Lock Loops (PLL)" - Navipedia

Here, we are looking at the prompt correlation (center). The PLL requires that all the signal's energy be on the real axis (I). If energy “leaks” onto the imaginary axis (Q), it means that our Doppler frequency is not perfect and that the phase is shifting. So we need to use the [Costas loop discriminator](https://gssc.esa.int/navipedia/index.php?title=Phase_Lock_Loop_(PLL)#:~:text=Insensitive%20to%20bit%20transitions%2C%20also%20called%20Costas%20loop)

$$E = \arctan\left(\frac{Q}{I}\right)$$
### Writing the discriminators

Now in go we can create a new file for those two discriminators and their states

```go
package gps  
  
import "math"  
  
type PLLState struct {  
    DopplerFreq float64 // The corrected frequency (NCO)  
    PhaseError  float64 // The error measured by the discriminator  
    Gain        float64 // The loop filter  
}  
  
type DLLState struct {  
    CodePhase float64 // The exact position in the code (NCO)  
    CodeError float64 // The measured error  
    Gain      float64 // The loop filter  
}  
  
func UpdatePLL(I, Q float64, state *PLLState) {  

	if I == 0 {  
		I = 1e-6 // Safety against division by zero  
	}
  
    // DISCRIMINATOR : Costas Loop  
    state.PhaseError = math.Atan(Q / I)  
  
    // LOOP FILTER & NCO UPDATE  
    state.DopplerFreq += state.PhaseError * state.Gain  
}  
  
func UpdateDLL(earlyPower, latePower float64, state *DLLState) {  
    // DISCRIMINATOR : Normalized Early-Minus-Late  
    sum := earlyPower + latePower  
  
    if sum > 0 {  
       state.CodeError = (earlyPower - latePower) / sum  
    } else {  
       state.CodeError = 0.0 // Safety against division by zero  
    }  
  
    // LOOP FILTER & NCO UPDATE  
    state.CodePhase += state.CodeError * state.Gain  
}
```

For the PLL, we use a high In-Phase value ($I=100$) and a small Quadrature leak ($Q=20$) to simulate a slight phase misalignment, proving the loop correctly increases the Doppler frequency to re-center the signal.

```go  
func TestUpdatePLL(t *testing.T) {  
    state := PLLState{  
       DopplerFreq: 1500.0,  
       Gain:        0.1,  
    }  
  
    // Simulate phase error (energy leaking into Q axis)  
    I := 100.0  
    Q := 20.0  
  
    // Run loop to simulate continuous tracking over 20ms  
    for ms := 0; ms < 20; ms++ {  
       UpdatePLL(I, Q, &state)  
    }  
  
    // Doppler must increase to compensate for the positive phase error  
    if state.DopplerFreq <= 1500.0 {  
       t.Errorf("PLL failed to correct Doppler. Got: %f", state.DopplerFreq)  
    }  
}  
```

For the DLL, setting the Early power ($120$) higher than the Late power ($80$) mimics a local code slightly lagging behind the satellite, verifying that the discriminator successfully speeds up the phase to catch up.

```go  
func TestUpdateDLL(t *testing.T) {  
    state := DLLState{  
       CodePhase: 500.0,  
       Gain:      0.05,  
    }  
  
    // Early > Late indicates local code is lagging behind the received signal  
    earlyPower := 120.0  
    latePower := 80.0  
  
    UpdateDLL(earlyPower, latePower, &state)  
  
    // Math check: Error = (120 - 80) / 200 = 0.2  
    // Correction = 0.2 * 0.05 = 0.01    expectedPhase := 500.01  
    expectedPhase := 500.01
  
    if math.Abs(state.CodePhase-expectedPhase) > 0.0001 {  
       t.Errorf("DLL correction failed. Expected: %f, Got: %f", expectedPhase, state.CodePhase)  
    }  
}
```

```shell
$ make test                                                                    
go test -v -race ./...
...
=== RUN   TestPLL_Convergence
--- PASS: TestPLL_Convergence (0.00s)
=== RUN   TestDLL_Correction
--- PASS: TestDLL_Correction (0.00s)
PASS
ok      github.com/dayio/gps-decoder/internal/gps       1.132s
...
```

And our test will pass as expected, so now we have all cards in our hands to make the state machine.

Here is the `channel.go` that will implement the PRN state machine for switching between acquisition and tracking (and in the next article: decoding)

```go
package gps  
  
import (  
    "fmt"  
)  
  
type ChannelState int  
  
const (  
    StateAcquiring ChannelState = iota  
    StateTracking  
)  
  
func RunChannel(prn int, sampleStream <-chan []complex128, sampleRate float64) {  
    state := StateAcquiring  
    var trackState TrackingState  
  
    for chunk := range sampleStream {  
       switch state {  
  
       case StateAcquiring:  
          acquireResult := AcquireFFT(chunk, prn, sampleRate)  
  
          // Threshold to switch to tracking  
          if acquireResult.SNR > 6.0 {  
             fmt.Printf("[PRN %02d] Signal acquired - SNR: %.2f. Switching to TRACKING\n", prn, acquireResult.SNR)  
  
             trackState = TrackingState{  
                PRN:          prn,  
                CarrierPhase: 0.0,  
                PLL: PLLState{  
                   DopplerFreq: acquireResult.BestDoppler,  
                   Gain:        0.25,  
                },  
                DLL: DLLState{  
                   CodePhase: float64(acquireResult.BestPhase),  
                   Gain:      0.05,  
                },  
             }  
             state = StateTracking  
          }  
  
       case StateTracking:  
          trackResult := TrackChunk(chunk, &trackState, sampleRate)  
  
          // Loss of Lock detection  
          if trackResult.PromptPower < 50.0 {  
             fmt.Printf("[PRN %02d] Tracking lost - Fallback to ACQUISITION\n", prn)  
             state = StateAcquiring  
             continue  
          }  
  
          // Data Extraction  
          if trackResult.IsLocked {  
             // TODO: We will build this in the next article !  
          }  
       }  
    }  
}
```

And here is the `tracking.go` that will run when the PRN is in tracking mode, it will continuously correlate, test, and update the phase and Doppler frequency to keep a lock using our PLL and DLL functions.

```go
package gps  
  
import (  
    "math"  
    "math/cmplx"
)  
  
type TrackStepResult struct {  
    PromptPower float64  
    I           float64  
    IsLocked    bool  
}  
  
type TrackingState struct {  
    PRN          int  
    CarrierPhase float64 // Accumulated carrier phase  
    PLL          PLLState  
    DLL          DLLState  
}  
  
func TrackChunk(chunk []complex128, state *TrackingState, sampleRate float64) TrackStepResult {  
    // Spacing for Early and Late correlators (0.5 chip is standard)  
    codeSpacing := 0.5  
  
    // CORRELATE (Integrate & Dump)  
    // We test the local code at three distinct positions    early := Correlate(chunk, state.PRN, state.DLL.CodePhase-codeSpacing, state.PLL.DopplerFreq, state.CarrierPhase, sampleRate)  
    prompt := Correlate(chunk, state.PRN, state.DLL.CodePhase, state.PLL.DopplerFreq, state.CarrierPhase, sampleRate)  
    late := Correlate(chunk, state.PRN, state.DLL.CodePhase+codeSpacing, state.PLL.DopplerFreq, state.CarrierPhase, sampleRate)  
  
    // DLL UPDATE (Fix Code Delay)  
    earlyPower := cmplx.Abs(early)  
    latePower := cmplx.Abs(late)  
    UpdateDLL(earlyPower, latePower, &state.DLL)  
  
    // PLL UPDATE (Fix Doppler Frequency)  
    I := real(prompt)  
    Q := imag(prompt)  
    UpdatePLL(I, Q, &state.PLL)  
  
    // CARRIER PHASE ADVANCE  
    // We move the carrier phase forward by 1ms for the next chunk    state.CarrierPhase += 2.0 * math.Pi * state.PLL.DopplerFreq * 0.001  
    state.CarrierPhase = math.Mod(state.CarrierPhase, 2*math.Pi) // Keep it within 0 to 2PI as the phase rotates  
  
    // RETURN METRICS    // If the phase error is small enough, we consider the PLL "locked"    isLocked := math.Abs(state.PLL.PhaseError) < 0.2  
  
    return TrackStepResult{  
       PromptPower: cmplx.Abs(prompt),  
       I:           I,  
       IsLocked:    isLocked,  
    }  
}  
  
func Correlate(chunk []complex128, prn int, codePhase, doppler, initialPhase, sampleRate float64) complex128 {  
    goldCode := GenerateGoldCode(prn)  
    var correlation complex128  
  
    for i, sample := range chunk {  
  
       // Calculate the Gold Code for this specific sample  
       chipIdx := int(codePhase+float64(i)*(1023000.0/sampleRate)) % 1023  
  
       if chipIdx < 0 {  
          chipIdx += 1023  
       }  
  
       // Generate the local carrier wave to cancel the Doppler  
       angle := initialPhase + 2.0*math.Pi*doppler*(float64(i)/sampleRate)  
  
       // Negative angle to untwist the signal  
       phasor := cmplx.Exp(complex(0, -angle))  
  
       // Multiply incoming signal by local code and local carrier  
       correlation += sample * complex(goldCode[chipIdx], 0) * phasor  
    }  
  
    return correlation  
}
```

Now if we run the code, we can witness the acquisition, the tracking and the loss of signal and corrections to lock again. The output below ran for a minute, and each dot is a chunk being fed to the 32 `RunChannel` functions running concurrently.

```shell
$ make run
...........
[PRN 21] Signal acquired - SNR: 8.10. Switching to TRACKING
[PRN 15] Signal acquired - SNR: 11.46. Switching to TRACKING
[PRN 05] Signal acquired - SNR: 7.33. Switching to TRACKING
[PRN 22] Signal acquired - SNR: 6.74. Switching to TRACKING
[PRN 14] Signal acquired - SNR: 9.72. Switching to TRACKING
[PRN 20] Signal acquired - SNR: 8.62. Switching to TRACKING
[PRN 30] Signal acquired - SNR: 7.39. Switching to TRACKING
.
[PRN 23] Signal acquired - SNR: 6.89. Switching to TRACKING
.
[PRN 24] Signal acquired - SNR: 6.03. Switching to TRACKING
.......................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
[PRN 24] Tracking lost - Fallback to ACQUISITION
..
[PRN 24] Signal acquired - SNR: 6.34. Switching to TRACKING
...........................
[PRN 05] Tracking lost - Fallback to ACQUISITION
.[PRN 05] Signal acquired - SNR: 7.02. Switching to TRACKING
................................................................................................................
[PRN 15] Tracking lost - Fallback to ACQUISITION
.
[PRN 15] Signal acquired - SNR: 9.71. Switching to TRACKING
.........................................................................................................................................................
[PRN 05] Tracking lost - Fallback to ACQUISITION
.
[PRN 05] Signal acquired - SNR: 6.06. Switching to TRACKING
........................................................................
```

And as we can see, the signal evolves, and the lock is being regained pretty quickly due to the DLL and PLL loop filters smoothing out the signal, also moving the signal by 1ms for each next chunk really helps.

> Disclaimer: For simplicity, we are using a first-order loop filter here with a simple gain. A production-grade receiver would use a second-order PI filter (Proportional Integral) to track phase acceleration.

## Listening to the Satellite (Data Extraction)

Well, I'll be nice, let's decode a signal to see how it goes before we continue in the next article !

First we'll create a `navigation_data.go` file with a `ProcessNavigationData` to decode a bit. Since the GPS navigation message travels at 50 bits per second, each bit lasts exactly 20 ms. Because our TrackChunk function processes the signal in 1-ms blocks, we simply need to accumulate 20 consecutive blocks to determine the polarity of a single bit.

### Where is 0x8B ?

Also for this example we will stick to one PRN (15) and will try to find the *Preamble* (start of packet identifier as described in article one) that can be normal `10001011 (0x8B)` or inverted `01110100 (0x74)` because the PLL can track the signal at 0 and 180°.

```go
package gps  
  
import "fmt"  
  
type NavDataState struct {  
    BufferCount int     // Counts up to 20ms (20 tracking outputs)  
    Accumulator float64 // Integrates the In-Phase (I) energy over 20ms  
    ShiftRegister uint8 // Our 8-bit shift register (to find the preamble)
}  
  
func ProcessNavigationData(prn int, I float64, state *NavDataState) {
	if prn != 15 { return }

	state.Accumulator += I
	state.BufferCount++

	if state.BufferCount >= 20 {
	
		// Determine the bit
		bit := uint8(0)
		if state.Accumulator < 0 {
			bit = 1
		}

		// Push the bit into shift register
		state.ShiftRegister = (state.ShiftRegister << 1) | bit

		// Check for Preamble
		if state.ShiftRegister == 0x8B { // 0001011 (0x8B
			fmt.Printf("\n[PRN %02d] Preamble found (Normal) \n", prn)
		} else if state.ShiftRegister == 0x74 { // 01110100 (0x74)
			fmt.Printf("\n[PRN %02d] Preamble found (Inverted) \n", prn)
		}

		fmt.Printf("%d", bit)

		state.BufferCount = 0
		state.Accumulator = 0.0
	}
}
```

And in the `channel.go` we can now call our function and also declare a local state to store our bytes

```go
  
func RunChannel(prn int, sampleStream <-chan []complex128, sampleRate float64) {  
    
    state := StateAcquiring  
    
    var trackState TrackingState  
  
    navState := NavDataState{}  // New data state
  
    for chunk := range sampleStream {  
       switch state {  
  
       case StateAcquiring:   
			...
       case StateTracking:  
          trackResult := TrackChunk(chunk, &trackState, sampleRate)  
  
          if trackResult.PromptPower < 50.0 {  
             state = StateAcquiring  
             continue  
          }  
  
          // Data Extraction  
          if trackResult.IsLocked {  
             ProcessNavigationData(prn, trackResult.I, &navState) // Decode 20ms of data
          }  
       }  
    }
}
```

```shell
$ make run
go build -o bin/gps-decoder cmd/gps-decoder/main.go
./bin/gps-decoder
10111001101111111110110011110001010010101000011100100010100101000011010111010
[PRN 15] Preamble found (Inverted) 
00001001100101001100111100111010
[PRN 15] Preamble found (Inverted) 
011110100111111111110100011011001111001100100010001001001000101
[PRN 15] Preamble found (Normal) 
1011110111101110010001101010110011001101101110110011011000110010100011101100000000100110111110111010
[PRN 15] Preamble found (Inverted) 
00111010
[PRN 15] Preamble found (Inverted) 
00111010
[PRN 15] Preamble found (Inverted) 
0010100110110101100101001011010010011010010100000100111011110110101100000000101000100100101100110111010
[PRN 15] Preamble found (Inverted) 
0001110111110000111110000000011111111111000110110010011110000010010110000101011110101110010100011011111110110011000101
[PRN 15] Preamble found (Normal) 
101101110110101001111011100111000111100011101011100011010000010011001000100100001001001000000110011100000001001110111010
```

## Conclusion

And after running, we can see that we have the 0x8B preamble coming. This short 8-bit message is the "Hello World" (literally) of space. It confirms that PRN 15 is communicating with us and that we are ready to translate its binary frames into GPS coordinates.

In the next article we will decode those frames and talk about Frame Alignment, Bit Sync, Parity Check, and Ephemeris. We will also make use of the data to locate ourselves, or as a starter, to get the current GPS time.

All the code is [available on Github](https://github.com/dayio/gps-decoder), explore the commits to see the evolution, the refactoring, and performance improvements.

Interesting sources:
- [Tracking Loops](https://gssc.esa.int/navipedia/index.php/Tracking_Loops)
- [Delay Lock Loop (DLL)](https://gssc.esa.int/navipedia/index.php?title=Delay_Lock_Loop_(DLL))
- [Phase Lock Loop (PLL)](https://gssc.esa.int/navipedia/index.php?title=Phase_Lock_Loop_(PLL))
- [Correlators](https://gssc.esa.int/navipedia/index.php?title=Correlators)
- [Understanding GPS/GNSS: Principles and Applications](https://api.pageplace.de/preview/DT0400.9781630814427_A37804737/preview-9781630814427_A37804737.pdf) (You can find the full book online)
