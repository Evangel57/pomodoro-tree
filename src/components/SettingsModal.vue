<script setup>
import { ref } from 'vue'

const props = defineProps({
  settings: { type: Object, required: true }
})

const emit = defineEmits(['save', 'close'])

const local = ref({ ...props.settings })

function save() {
  emit('save', { ...local.value })
}
</script>

<template>
  <div class="overlay" @click.self="emit('close')">
    <div class="modal">
      <div class="modal-header">
        <h2>Settings</h2>
        <button class="close-btn" @click="emit('close')">✕</button>
      </div>

      <div class="fields">
        <div class="field">
          <label>Focus duration (minutes)</label>
          <input type="number" v-model.number="local.workMinutes" min="1" max="120" />
        </div>
        <div class="field">
          <label>Break duration (minutes)</label>
          <input type="number" v-model.number="local.breakMinutes" min="1" max="60" />
        </div>
        <div class="field">
          <label>Sessions to grow full tree</label>
          <input type="number" v-model.number="local.sessionsToFullTree" min="1" max="24" />
        </div>
      </div>

      <div class="modal-actions">
        <button class="btn-cancel" @click="emit('close')">Cancel</button>
        <button class="btn-save" @click="save">Save</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
  backdrop-filter: blur(4px);
}

.modal {
  background: var(--bg2);
  border: 1px solid var(--border);
  border-radius: 20px;
  padding: 28px;
  width: 100%;
  max-width: 380px;
  box-shadow: var(--shadow);
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 24px;
}

.modal-header h2 {
  font-size: 18px;
  font-weight: 700;
}

.close-btn {
  background: var(--bg3);
  border: none;
  border-radius: 8px;
  padding: 6px 10px;
  cursor: pointer;
  color: var(--text2);
  font-size: 14px;
}

.fields { display: flex; flex-direction: column; gap: 16px; }

.field { display: flex; flex-direction: column; gap: 6px; }

label {
  font-size: 13px;
  color: var(--text2);
  font-weight: 500;
}

input {
  background: var(--bg3);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 10px 14px;
  font-size: 15px;
  color: var(--text);
  outline: none;
  transition: border-color 0.2s;
}
input:focus { border-color: var(--accent); }

.modal-actions {
  display: flex;
  gap: 10px;
  margin-top: 24px;
  justify-content: flex-end;
}

.btn-cancel {
  background: var(--bg3);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 10px 20px;
  color: var(--text2);
  cursor: pointer;
  font-size: 14px;
}

.btn-save {
  background: var(--accent);
  border: none;
  border-radius: 10px;
  padding: 10px 24px;
  color: white;
  font-weight: 600;
  cursor: pointer;
  font-size: 14px;
  transition: opacity 0.2s;
}
.btn-save:hover { opacity: 0.9; }

.tree-options { display: flex; gap: 8px; }
.tree-opt {
  flex: 1;
  background: var(--bg3);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 9px 6px;
  font-size: 13px;
  cursor: pointer;
  color: var(--text2);
  transition: all 0.15s;
}
.tree-opt:hover { color: var(--text); }
.tree-opt.active { border-color: var(--accent); color: var(--accent); background: var(--bg3); }
</style>
