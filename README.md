# Simple Equalizer
#### Video Demo:  <[URL HERE>](https://youtu.be/LbnM76WcvRo?si=VaUFqAcCwEtMaZ6s)
#### Description:
Simple Equalizer is an audio plug-in designed to manipulate the frequency of a given audio file (.wav/.aiff). As a composer, I developed this plug-in to explore the process of creating audio tools and enhance my understanding of audio plug-in development.

## Features

### 1. Frequency Manipulation
   - **Low Cut:** Cut the low-frequency range.
   - **High Cut:** Cut the high-frequency range.
   - **Peak:** Highlight frequencies in between.

### 2. Peak Feature Enhancements
   - **Peak Gain:** Adjust the gain of the selected frequency point.
   - **Peak Quality:** Fine-tune the quality of the selected frequency point.

### 3. Slope Options
   - Choose from various options for low-cut and high-cut slopes to customize the steepness of the cut line.

## Development

To develop this audio tool, the [JUCE](https://juce.com/) framework was used. JUCE is a comprehensive framework for developing audio applications from scratch. The development environment utilized was Xcode on macOS.

### Getting Started
1. Clone the repository.

2. Open the project in Xcode.

3. Build and run the project.

### Usage
1. Import an audio file (.wav and .aiff only) from:
   Plugins -> Create Plug-in -> Apple -> AUAudioFilePlayer

2. Import the Equalizer plug-in from:
   Plugins -> Create Plug-in -> yourcompany -> SimpleEQ

3. Connect Audio File Player and SimpleEQ to Audio Output as follows:
   AUAudioFilePlayer -> SimpleEQ(VST3) -> Audio Output(internal)
 
4. Utilize the low-cut, high-cut, and peak features to manipulate the frequency.

5. Fine-tune the selected frequency point using the Peak Gain and Peak Quality parameters.

6. Choose from various slope options for low-cut and high-cut features.

7. Save your adjusted setup by command + S.
 
### Project Walkthrough

## Parameter Setup

Audio plugins rely on parameters to control various aspects of the Digital Signal Processing (DSP). JUCE introduces the `AudioProcessorValueTreeState` object to synchronize these parameters with the GUI controls and DSP variables. In `<PluginProcessor.h>`, we declare and initialize this object:

```cpp
juce::AudioProcessorValueTreeState apvts {*this, nullptr, "Parameters", createParameterLayout()};
```

The `createParameterLayout` function is crucial, providing a parameter layout for the `AudioProcessorValueTreeState`:

```cpp
static juce::AudioProcessorValueTreeState::ParameterLayout createParameterLayout();
```

Parameters for the Low Cut, High Cut, and Peak bands are set in `<PluginProcessor.cpp>` using the `AudioProcessorParameter` class. Various parameters such as frequency cutoff, slope, center frequency, gain, and quality are configured. The choice of using `AudioParameter<Float>` and `AudioParameter<choice>` depends on the nature of the parameter.

The range and initial values for parameters like LowCut Freq, HighCut Freq, and Peak Freq are set to 20Hz to 20kHz, with default values of 20Hz, 20kHz, and 750Hz, respectively. Peak Gain is expressed in decibels, ranging from -24 to 24. Peak Quality controls the width of the peak band, with a range from 0.1 to 10. Both Peak Gain and Peak Quality default to 1.

For the LowCut Slope and HighCut Slope, options are limited to 4 choices (12, 24, 36, and 48 decibels per octave).

After setting up parameters in the layout, they are passed to the `AudioProcessorValueTreeState` constructor:

```cpp
juce::AudioProcessorValueTreeState apvts {*this, nullptr, "Parameters", createParameterLayout()};
```

## GUI Interface

As the UI design is beyond the project scope, a JUCE template interface is used. The template is set up in the `createEditor` function:

```cpp
juce::AudioProcessorEditor* SimpleEQAudioProcessor::createEditor()
{
    return new juce::GenericAudioProcessorEditor (*this);
}
```

## Setting up the DSP

JUCE's DSP namespace is designed for processing mono audio by default. To process stereo channels, such as in this plugin, we need to duplicate the required components. Type aliases are created to simplify complex namespace and template definitions. A `Filter` alias is introduced for the IIR filter:

```cpp
using Filter = juce::dsp::IIR::Filter<float>;
```

A `CutFilter` is defined for a 48 dB/octave response:

```cpp
using CutFilter = juce::dsp::ProcessorChain<Filter, Filter, Filter, Filter>;
```

To represent the entire mono signal path, a `MonoChain` is created:

```cpp
using MonoChain = juce::dsp::ProcessorChain<CutFilter, Filter, CutFilter>;
```

Two instances of `MonoChain`, `leftChain`, and `rightChain`, are declared for the left and right channels.

Before using these chains, they need to be prepared in the `prepareToPlay` function:

```cpp
void SimpleEQAudioProcessor::prepareToPlay(double sampleRate, int samplesPerBlock)
{
    juce::dsp::ProcessSpec spec;
    spec.maximumBlockSize = samplesPerBlock;
    spec.numChannels = 1;
    spec.sampleRate = sampleRate;
    leftChain.prepare(spec);
    rightChain.prepare(spec);
}
```

In the `processBlock` function, audio is processed through the chains using `ProcessContext`:

```cpp
void SimpleEQAudioProcessor::processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    juce::dsp::AudioBlock<float> block(buffer);
    auto leftBlock = block.getSingleChannelBlock(0);
    auto rightBlock = block.getSingleChannelBlock(1);
    juce::dsp::ProcessContextReplacing<float> leftContext(leftBlock);
    juce::dsp::ProcessContextReplacing<float> rightContext(rightBlock);
    leftChain.process(leftContext);
    rightChain.process(rightContext);
}
```

## Connecting the Parameters

A data structure, `ChainSettings`, is introduced to represent all parameter values:

```cpp
struct ChainSettings
{
    float peakFreq{ 0 }, peakGainInDecibels{ 0 }, peakQuality { 1.f };
    float lowCutFreq{ 0 }, highCutFreq{ 0 };
    Slope lowCutSlope { Slope::Slope_12}, highCutSlope { Slope::Slope_12 };
};
```

A helper function, `getChainSettings`, provides all parameter values in this data structure:

```cpp
ChainSettings getChainSettings(juce::AudioProcessorValueTreeState& apvts)
{
    ChainSettings settings;
    // ... (assign parameter values)
    return settings;
}
```

Various functions (`updatePeakFilter`, `updateCoefficients`, `updateLowCutFilters`, `updateHighCutFilters`) are created to update the filters based on the parameter settings. These functions are triggered from the `updateFilters` function:

```cpp
void SimpleEQAudioProcessor::updateFilters()
{
    auto chainSettings = getChainSettings(apvts);
    updateLowCutFilters(chainSettings);
    updatePeakFilter(chainSettings);
    updateHighCutFilters(chainSettings);
}
```

## Conclusion

This comprehensive walkthrough has covered the parameter setup, DSP configuration, and parameter connection for the SimpleEQ audio plugin. Understanding each step is crucial for developers looking to customize and enhance the plugin for specific audio processing requirements. The integration of JUCE's powerful features ensures the effectiveness and versatility of the SimpleEQ plugin in audio manipulation.

