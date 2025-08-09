from mido import Message, MidiFile, MidiTrack, MetaMessage, bpm2tempo

# =========================
# Configuración del proyecto
# =========================
BPM = 126
TPB = 480  # ticks por negra (resolución)
TEMPO = bpm2tempo(BPM)

# Utilidades de tiempo
def beats(n):       # n negras
    return int(n * TPB)

def bars(n):        # n compases en 4/4
    return int(n * 4 * TPB)

# Notas MIDI (G# menor)
NOTE = {
    'G#1': 32, 'F#1': 30,
    'G#2': 44, 'D#3': 51, 'F#3': 54, 'G#3': 56
}

# =========================
# Crear archivo y pistas
# =========================
mid = MidiFile(ticks_per_beat=TPB)

# --------- Track 1: PAD OSCURO (con CC74 para filtro) ----------
pad = MidiTrack()
mid.tracks.append(pad)

# Meta: tempo y compás
pad.append(MetaMessage('set_tempo', tempo=TEMPO, time=0))
pad.append(MetaMessage('time_signature', numerator=4, denominator=4, time=0))

# Plan: 8 compases de break
# Compases 1-4:  G#2 (sostenido)
# Compases 5-6:  D#3 (sostenido) + CC74 abriendo
# Compás   7:    F#3
# Compás   8:    G#3
#
# Además, mandamos CC74 en compases 5-8 para abrir filtro (valores escalonados).
BAR = bars(1)

# Compases 1-4: G#2
pad.append(Message('note_on', note=NOTE['G#2'], velocity=80, time=0))
pad.append(Message('note_off', note=NOTE['G#2'], velocity=80, time=bars(4)))

# Inicio compás 5: CC74 (filtro) valor bajo + D#3 ON
pad.append(Message('control_change', control=74, value=30, time=0))  # filtro cerrado-ish
pad.append(Message('note_on', note=NOTE['D#3'], velocity=80, time=0))

# Compás 6 (después de 1 compás): CC74 sube un poco
pad.append(Message('control_change', control=74, value=55, time=BAR))

# Fin compás 6: D#3 OFF (tras 2 compases)
pad.append(Message('note_off', note=NOTE['D#3'], velocity=80, time=BAR))

# Compás 7: F#3 ON
pad.append(Message('note_on', note=NOTE['F#3'], velocity=80, time=0))

# Mitad compás 7: CC74 sube más
pad.append(Message('control_change', control=74, value=80, time=beats(2)))

# Fin compás 7: F#3 OFF
pad.append(Message('note_off', note=NOTE['F#3'], velocity=80, time=beats(2)))

# Compás 8: G#3 ON
pad.append(Message('note_on', note=NOTE['G#3'], velocity=80, time=0))

# Primer cuarto de compás 8: CC74 casi abierto
pad.append(Message('control_change', control=74, value=100, time=beats(1)))

# Resto compás 8: G#3 OFF al final del compás
pad.append(Message('note_off', note=NOTE['G#3'], velocity=80, time=beats(3)))

# --------- Track 2: BAJO MINIMAL "GHOST" ----------
bass = MidiTrack()
mid.tracks.append(bass)

# Meta: tempo (por compatibilidad)
bass.append(MetaMessage('set_tempo', tempo=TEMPO, time=0))
bass.append(MetaMessage('time_signature', numerator=4, denominator=4, time=0))

# Notas de bajo: golpes cortos (1/4) en compases 1, 3, 5 y 7
# Patrón: G#1 (compás 1), F#1 (compás 3), G#1 (compás 5), F#1 (compás 7)
# Deja silencios grandes para tensión.
def add_bass_hit(track, bar_index, note_name):
    # Mueve hasta el inicio del compás correspondiente (desde donde estemos)
    # Calculamos el delta necesario desde el último evento
    # Para simplificar, vamos a avanzar desde la posición actual con el tiempo relativo deseado.
    track.append(Message('note_on', note=NOTE[note_name], velocity=90, time=0))
    track.append(Message('note_off', note=NOTE[note_name], velocity=90, time=beats(1)))  # 1/4 de compás

# Programación del tiempo en el bass track:
# Vamos a "caminar" por los 8 compases y disparar donde toca,
# usando tiempos relativos para llegar a cada compás.

current_time = 0

def advance(track, ticks):
    # Insertamos un evento "no-op" desplazando el tiempo del siguiente evento.
    # Lo haremos poniendo el time del próximo evento real.
    # En Mido, el tiempo se cuenta en el siguiente mensaje, así que
    # usaremos un Note On con velocity 0 como "espera"? No hace falta: simplemente
    # el próximo evento que añadamos tendrá 'time=ticks'.
    # Para implementarlo de forma limpia, devolvemos 'ticks' y el próximo
    # evento debe usarlo como 'time'.
    return ticks

# Compás 1: golpe G#1 en el inicio del compás
bass.append(Message('note_on', note=NOTE['G#1'], velocity=90, time=0))
bass.append(Message('note_off', note=NOTE['G#1'], velocity=90, time=beats(1)))

# Avanzar hasta inicio compás 3: faltan compás 1 resto (3/4) + compás 2 completo (4/4) = 7/4
bass.append(Message('control_change', control=1, value=0, time=beats(7)))  # dummy CC to consumir tiempo

# Compás 3: golpe F#1
bass.append(Message('note_on', note=NOTE['F#1'], velocity=90, time=0))
bass.append(Message('note_off', note=NOTE['F#1'], velocity=90, time=beats(1)))

# Avanzar hasta inicio compás 5: resto compás 3 (3/4) + compás 4 (4/4) = 7/4
bass.append(Message('control_change', control=1, value=0, time=beats(7)))

# Compás 5: golpe G#1
bass.append(Message('note_on', note=NOTE['G#1'], velocity=90, time=0))
bass.append(Message('note_off', note=NOTE['G#1'], velocity=90, time=beats(1)))

# Avanzar hasta inicio compás 7: resto compás 5 (3/4) + compás 6 (4/4) = 7/4
bass.append(Message('control_change', control=1, value=0, time=beats(7)))

# Compás 7: golpe F#1
bass.append(Message('note_on', note=NOTE['F#1'], velocity=90, time=0))
bass.append(Message('note_off', note=NOTE['F#1'], velocity=90, time=beats(1)))

# Avanzar hasta final del compás 8 (por si quieres que el clip dure 8 compases exactos):
# Faltan resto compás 7 (3/4) + compás 8 (4/4) = 7/4
bass.append(Message('control_change', control=1, value=0, time=beats(7)))

# =========================
# Guardar archivo
# =========================
out_name = "bajada_oscura_minimal.mid"
mid.save(out_name)
print(f"Creado: {out_name} — 2 pistas (Pad + Bajo), 126 BPM, G# menor")
