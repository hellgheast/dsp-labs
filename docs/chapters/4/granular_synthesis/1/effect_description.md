# 4.1 Granular synthesis pitch shifting, _explained_

We will briefly describe the key components for pitch shifting with granular synthesis so that it is
 clear what components we will have to implement. A more in-depth and intuitive explanation 
 (with audio examples and visuals) can be found on Paolo Prandoni's
  [IPython notebook](http://nbviewer.jupyter.org/github/prandoni/COM303/blob/master/voice_transformer/voicetrans.ipynb).

## <a id="alter_pitch"></a>Resampling to alter pitch

In the simple alien voice effect, we were able to shift up the entire spectrum of the signal by multiplying the input signal with a sinusoid. This is easy to see if we consider the Fourier Transform.

$$
x(t) \sin(2\pi f_{mod}t) \xrightarrow{\mathscr{F}} \frac{1}{2j}[X(f-f_{mod}) - X(f+f_{mod})].
$$

However, due to our limited sampling frequency the higher frequencies would _wrap around_ to the
 lower frequencies, i.e. alias. This distortion is what contributes to our alien voice effect, but if $$ f_{mod} $$ is too high, intelligibility will be very poor.

It is also possible to alter the pitch without introducing aliasing artifacts that will significantly affect the intelligibility. This can be done by _resampling_ a signal and playing it at the original sampling rate.

For example, in order to shift the pitch up, we would subsample the original samples so that the 
original signal is effectively played at a faster rate when the subsampled signal is played at the original rate. Conversely, for shifting the pitch down, we could add extra samples in between the original samples so that the original signal is effectively at a slower rate when played at the original rate.

In order to support arbitrary pitch shifts, we will need to access values at non-integer samples, i.e. fractional samples. A simple solution (and the one we will use) is to linearly interpolate. Given a discrete signal $$ x[n] $$ and a real-valued time index $$ N \le t \le N+1 $$, we can compute an approximate value for $$ x(t) $$ as such:

<a id="interp_times"></a>
$$
\hat{x}(t) = ax[N] + (1-a) x[N+1],
$$

where $$ N $$ is the _largest_ integer smaller than $$ t $$ and:

<a id="interp_amps"></a>
$$
a = 1 - (t-N).
$$

With this simple interpolation, we can approximate our signal for arbitrary time values. Moreover, due to its simplicity the interpolation can be implemented efficiently in real-time!

Unfortunately, the resampling operation itself changes the duration of our signal, which makes it impossible to implement in real-time as we do not have access to all samples at once.

## <a id="grains"></a>Grains

To this end, we will be using the idea of [granular synthesis](https://en.wikipedia.org/wiki/Granular_synthesis) to ensure our final output signal is of the same duration. The idea is to concatenate very short sound snippets (1-50 ms) called "grains" to form a new waveform. By varying the content of the grains and adjusting their rates, one can creatively add new _timbres_ to produce sounds of musical interest!

For example, we can artificially double the length of our signal, by taking grains of 32 ms and concatenating them one after the other. This doubling can be seen in the figure below where it is applied to a simple sinusoid signal.

<div style="text-align:center"><img src ="figs/doubling_discontinuity.png" width="600"/></div>
<br>

_Figure: Doubling signal with grain length of 32 ms (without windowing). Red-dotted line indicate 
the boundary between the same grain that is doubled. Green-dotted lines indicate the beginning of 
a new grain._

At the intersection between doubled grains (red-dotted line) in the figure above, we can observe an undesired discontinuity. To deal with this discontinuity, we need to _crossfade_ between grains by applying a _tapered_ window that rounds off to zero at the beginning and the end of each grain. An example window is shown below.

<div style="text-align:center"><img src ="figs/taper_window.png" width="600"/></div>
<br>

_Figure: Tapering window to crossfade between grains._

Now we need to align/overlap the grains so that the sum of their tapered windows adds to 1 as shown in the figure below.

<div style="text-align:center"><img src ="figs/windows_overlap.png" width="600"/></div>
<br>

Now when we double grains, we will have to overlap them by a certain amount as seen in the figure above. We will refer to the length between the beginning of one tapered window to the beginning of another tapered window as the **_stride_**. Moreover, the stride will be our frame length (number of samples per channel to process per buffer) when we apply our algorithm in real-time. Let's apply this tapered window on our sinusoid from before to make sure it removes the discontinuities.

<div style="text-align:center"><img src ="figs/doubling_continuous.png" width="600"/></div>
<br>

_Figure: Windowed sinusoid grains. Middle plot contains overlapping grains before tapering._

We can observe a tradeoff in choosing the length of this overlap factor (0-100%) with adjacent grains. An overlap close 0% will bring us towards our discontinuous signal while an overlap close to 100% will result in a shorter stride length, _giving us less time to process our buffer_.

A similar statement can be said about the chosen length for the grains. A short grain gives less time for processing while having lower latency; while a long grain will give us more time to process but increase the latency.

## <a id="combining"></a>Combining resampling with grains

Now that we know how to artificially increase the length of a signal with grains, we can combine this with resampling so that our input and output signals have the same duration. Thus, allowing us to perform pitch shifting! We recommend checking out the [IPython notebook](http://nbviewer.jupyter.org/github/prandoni/COM303/blob/master/voice_transformer/voicetrans.ipynb) for a visual representation of the resampled grains and additional explanations.



