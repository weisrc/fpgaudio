<template>
  <q-page padding class="q-gutter-y-md">
    <q-card flat bordered>
      <q-card-section horizontal>
        <q-card-section class="q-pt-xs q-gutter-y-sm">
          <div class="text-overline">Step 1</div>
          <div class="text-h5 q-mt-sm q-mb-xs">
            MIDI to Tone.js JSON conversion
          </div>
          <div class="text-caption text-grey">
            Since MIDI files are binary, we need to convert them to a more
            accessible format for JavaScript such as JSON as it is this frontend
            that is going to convert it back to a compact binary format for the
            FPGA. You can also play the MIDI file here thanks to Tone.js!
          </div>
          <br />
          <Editor
            height="50vh"
            theme="vs-dark"
            language="json"
            :value="json"
            v-if="json"
          />
        </q-card-section>

        <q-card-section class="col-5 flex flex-center">
          <q-img
            class="rounded-borders"
            src="https://images.unsplash.com/photo-1507838153414-b4b713384a76"
          />
        </q-card-section>
      </q-card-section>

      <q-separator />

      <q-card-actions class="justify-between">
        <q-file
          v-model="file"
          label="MIDI Audio File"
          accept=".mid"
          @update:model-value="onFileChange"
        >
          <template v-slot:before>
            <q-icon name="music_note" />
          </template>
          <template v-slot:append>
            <q-icon v-if="file !== null" name="close" class="cursor-pointer" />
          </template>
        </q-file>
        <div>
          <q-btn
            icon="play_arrow"
            @click="play"
            :disable="!json"
            v-if="!playing"
            >Play</q-btn
          >
          <q-btn icon="pause" @click="stop" :disable="!json" v-else>Stop</q-btn>
        </div>
      </q-card-actions>
    </q-card>

    <q-card flat bordered>
      <q-card-section horizontal>
        <q-card-section class="q-pt-xs q-gutter-y-sm">
          <div class="text-overline">Step 2</div>
          <div class="text-h5 q-mt-sm q-mb-xs">Verilog Code Generation</div>
          <div class="text-caption text-grey">
            What is the baes clock frequency? What should be the main note unit?
            Should we override the beats per minute? Should we override all note
            duration to give them a punch? Please enter the information on the
            side input field and click generate. The track encoding format is
            simple: every track is an array of bytes. If the value of the byte
            is less than 128, then the byte is an actual MIDI note that will
            play for unit time. However, if it is larger, the byte is a pause.
            The following 7 bytes determines how unit of time to pause on the
            track. This pretty much it!
          </div>
          <Editor
            height="50vh"
            theme="vs-dark"
            language="verilog"
            :value="code"
            v-if="code"
          />
        </q-card-section>

        <q-card-section class="flex flex-center col-3">
          <q-form @submit="generate" class="q-gutter-md">
            <q-input
              filled
              v-model="fclk"
              label="Clock Frequency (Hz)"
              type="number"
            />
            <q-select
              filled
              v-model="unit"
              :options="unitOptions"
              label="Note Unit"
            />
            <q-input
              filled
              v-model="bpmOverride"
              label="BPM Override"
              type="number"
            />
            <q-input
              filled
              v-model="durationOverride"
              label="Duration Override"
              type="number"
            />
            <q-btn
              label="Generate"
              type="submit"
              color="primary"
              :disable="!json || !fclk || !unit"
            />
          </q-form>
        </q-card-section>
      </q-card-section>
    </q-card>

    <q-card flat bordered>
      <q-card-section horizontal>
        <q-card-section class="q-pt-xs q-gutter-y-sm">
          <div class="text-overline">Step 3</div>
          <div class="text-h5 q-mt-sm q-mb-xs">Implement and have fun!</div>
          <div class="text-caption text-grey">
            You are very close to having your own MIDI synthesizer! All you need
            to do is implement the generated Verilog code in your FPGA and
            connect the audio output to your speakers. You can also use the
            generated JSON file to play the MIDI file in your browser.
          </div>
        </q-card-section>

        <q-card-section class="col-5 flex flex-center">
          <q-img
            class="rounded-borders"
            src="https://images.unsplash.com/photo-1514525253161-7a46d19cd819"
          />
        </q-card-section>
      </q-card-section>
    </q-card>
  </q-page>
</template>

<script lang="ts">
import { computed, defineComponent, ref } from 'vue';
import Editor from '@guolao/vue-monaco-editor';
import { Midi, MidiJSON } from '@tonejs/midi';
import { now, PolySynth } from 'tone';
import { parseMidi } from 'midi-file';

const synths: PolySynth[] = [];

export default defineComponent({
  name: 'IndexPage',
  components: {
    Editor,
  },
  setup() {
    const file = ref<File>();
    const midi = ref<MidiJSON>();
    const playing = ref(false);
    const fclk = ref(10e6);
    const code = ref('');
    const minVelocity = ref(20);
    const unit = ref({ label: '16th', value: 16 });
    const bpmOverride = ref<number>();
    const durationOverride = ref<number>();

    return {
      file,
      fclk,
      playing,
      minVelocity,
      code,
      unit,
      bpmOverride,
      durationOverride,
      unitOptions: [
        { label: '128th', value: 128 },
        { label: '64th', value: 64 },
        { label: '32th', value: 32 },
        { label: '16th', value: 16 },
        { label: '8th', value: 8 },
        { label: '4th', value: 4 },
        { label: '2nd', value: 2 },
        { label: '1st', value: 1 },
      ],
      json: computed(() => JSON.stringify(midi.value, null, 2)),
      async onFileChange() {
        if (!file.value) return;
        const buffer = await file.value.arrayBuffer();
        const mid = new Midi(buffer);
        console.log(parseMidi(new Uint8Array(buffer)));
        midi.value = mid.toJSON();
      },
      stop() {
        synths.forEach((synth) => synth.disconnect());
        synths.length = 0;
        playing.value = false;
      },
      play() {
        if (!midi.value) return;
        const currentMidi = midi.value;
        if (playing.value || !currentMidi) return;
        const t = now();
        currentMidi.tracks.forEach((track) => {
          const synth = new PolySynth().toDestination();
          synths.push(synth);
          track.notes.forEach((note) => {
            const { name, velocity, duration, time } = note;
            synth.triggerAttackRelease(name, duration, time + t, velocity);
          });
        });
        playing.value = true;
      },
      generate() {
        if (!midi.value) return;
        const tracks: {
          start: number;
          end: number;
          midi: number;
          duration: number;
        }[][] = [[]];

        // ticks per unit
        const tpu = (midi.value.header.ppq * 4) / unit.value.value;

        for (const mtrack of midi.value.tracks) {
          for (const note of mtrack.notes.sort((a, b) => a.ticks - b.ticks)) {
            const { ticks, durationTicks, midi } = note;
            const start = Math.round(ticks / tpu);
            const duration =
              durationOverride.value ||
              Math.max(1, Math.round(durationTicks / tpu));
            const end = start + duration;
            for (let i = 0; i < tracks.length; i++) {
              const track = tracks[i];
              const tail = track[track.length - 1];
              const overlaps = tail && tail.start >= start && tail.end <= end;
              const note = { start, end, midi, duration };
              if (overlaps) {
                if (i === tracks.length - 1) {
                  tracks.push([note]);
                  break;
                } else {
                  continue;
                }
              } else {
                track.push(note);
                break;
              }
            }
          }
        }

        const binaries = tracks.map((track) => {
          let t = 0;
          const notes: number[] = [];
          for (const note of track) {
            const { start, end, midi, duration } = note;
            let pause = start - t;
            while (pause > 0) {
              const p = Math.min(pause, 127);
              notes.push(128 + p);
              pause -= p;
            }
            for (let i = 0; i < duration; i++) {
              notes.push(midi);
            }
            t = end;
          }
          return notes;
        });

        const bpm =
          bpmOverride.value || midi.value.header?.tempos?.[0]?.bpm || 120;

        const unitHz = (bpm / 60 / 4) * unit.value.value;
        const uclkPeriodT = Math.round((fclk.value / unitHz) * 4);

        const midiPeriodTOctave = new Array(12)
          .fill(0)
          .map((_, i) => {
            const freq = 440 * Math.pow(2, (i - 69) / 12);
            return Math.round((fclk.value / freq) * 4);
          })
          .reverse();

        console.log(midiPeriodTOctave);

        code.value = `/*
Generated by https://weisrc.github.io/fpgaudio
for ${file.value?.name || 'unknown'}
*/

module fpgaudio_synth(
  input wire clk,
  input wire reset,
  output wire [${binaries.length - 1}:0] out
);

// unit clock
wire uclk;
fpgaudio_divider fd_uclk (clk, ${uclkPeriodT}, uclk);

// track data
${binaries
  .map((bin, i) => {
    const hex = bin
      .reverse()
      .map((n) => n.toString(16).padStart(2, '0'))
      .join('');
    const size = bin.length * 8;
    return `wire [${size - 1}:0] track${i} = ${size}'h${hex};`;
  })
  .join('\n')}

// track pauses
${binaries.map((_, i) => `wire [31:0] pause${i};`).join('\n')}

// track cursors
${binaries.map((_, i) => `wire [31:0] cursor${i};`).join('\n')}

// midi data
${binaries
  .map((_, i) => {
    const data = new Array(8)
      .fill(0)
      .map((_, j) => `track${i}[cursor${i}+${7 - j}]`)
      .join(',');
    return `wire [7:0] midi${i} = {${data}};`;
  })
  .join('\n')}

// track cursor modules
${binaries
  .map(
    (bin, i) =>
      `fpgaudio_cursor fc${i}(uclk, reset, midi${i}, ${
        bin.length * 8
      }, cursor${i}, pause${i});`
  )
  .join('\n')}

// track oscillators
${binaries
  .map((_, i) => {
    return `fpgaudio_oscillator fo${i}(clk, midi${i}[6:0], !pause${i}, out[${i}]);`;
  })
  .join('\n')}

endmodule

module fpgaudio_cursor(
  input wire clk,
  input wire reset,
  input wire [7:0] data,
  input wire [31:0] max,
  output integer cursor,
  output reg [6:0] pause
);

always @(posedge clk or posedge reset) begin
  if (reset) begin
    pause = 0;
    cursor = 0;
  end else begin
    if (cursor >= max) pause = 1;
    else if (pause) pause = pause - 1;
    else begin
      pause = data[7] ? data[6:0] : 0;
      cursor = cursor + 8;
    end
  end
end

endmodule

module fpgaudio_oscillator(
  input wire clk,
  input wire [6:0] midi,
  input wire enable,
  output wire out
);

// midi lowest octave value frequency divider periods
wire [383:0] octave = 384'h${midiPeriodTOctave
          .map((p) => p.toString(16).padStart(8, '0'))
          .join('')};
wire [8:0] i = (midi % 12) << 5;
wire [31:0] base_period = {${new Array(32)
          .fill(0)
          .map((_, j) => `octave[i+${31 - j}]`)
          .join(',')}};
wire [31:0] period = base_period >> (midi / 12);
wire internal_out;
fpgaudio_divider fd_midi(clk, period, internal_out);
assign out = enable ? internal_out : 1'b0;

endmodule

module fpgaudio_divider(
  input wire clk,
  input wire [31:0] period_t,
  output reg dclk
);

integer t = 0;
always @(posedge clk) begin
  t = t + 1;
  if (t >= period_t) begin
    t = 0;
    dclk = ~dclk;
  end
end

endmodule

`;

        console.log(binaries);
      },
    };
  },
});
</script>
