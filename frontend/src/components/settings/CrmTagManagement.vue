<template>
  <div class="crmtag-settings">
    <header class="settings-section-header">
      <p class="subtitle">
        Tag CRM là <strong>thẻ phân loại nội bộ</strong> do tổ chức tự định nghĩa (vd "VIP", "Đầu tư",
        "Quan tâm BĐS"). Gắn ngay trên thanh nhập chat. Khác với tag Zalo Real (sync từ Zalo client).
      </p>
    </header>

    <!-- Toolbar -->
    <div class="toolbar">
      <button class="toolbar-btn primary" @click="openCreate">
        <span>+</span> Thêm tag CRM
      </button>
      <button class="toolbar-btn" :disabled="recounting" @click="recount">
        <span v-if="recounting">⟳…</span>
        <span v-else>⟳ Cập nhật số liệu sử dụng</span>
      </button>
      <div class="toolbar-spacer"></div>
      <input v-model="filter" type="text" placeholder="Lọc theo tên / danh mục…" class="filter-input" />
    </div>

    <!-- Stats summary -->
    <div class="stats-row">
      <div class="stat-card">
        <div class="stat-num">{{ tags.length }}</div>
        <div class="stat-label">Tổng tag</div>
      </div>
      <div class="stat-card">
        <div class="stat-num">{{ activeCount }}</div>
        <div class="stat-label">Đang dùng (≥1 KH)</div>
      </div>
      <div class="stat-card">
        <div class="stat-num">{{ unusedCount }}</div>
        <div class="stat-label">Chưa dùng</div>
      </div>
      <div class="stat-card">
        <div class="stat-num">{{ categoryCount }}</div>
        <div class="stat-label">Danh mục</div>
      </div>
    </div>

    <!-- Loading / Empty -->
    <div v-if="loading && !tags.length" class="loading-state">Đang tải…</div>
    <div v-else-if="!tags.length" class="empty-state">
      <p>Chưa có tag CRM nào.</p>
      <button class="toolbar-btn primary" @click="openCreate">+ Tạo tag đầu tiên</button>
    </div>

    <!-- Grouped by category -->
    <div v-else class="tags-grouped">
      <section v-for="group in groupedTags" :key="group.category" class="tag-group">
        <h4 class="group-title">
          {{ group.category }}
          <span class="group-count">{{ group.tags.length }}</span>
        </h4>
        <div class="tag-grid">
          <div v-for="tag in group.tags" :key="tag.id" class="tag-card" :class="{ inactive: !tag.isActive }">
            <!-- Preview pill -->
            <div class="tag-preview-row">
              <span
                class="tag-preview"
                :style="`background:${tag.color}22;color:${tag.color};border-color:${tag.color}`"
              >
                <span v-if="tag.emoji">{{ tag.emoji }} </span>{{ tag.name }}
              </span>
              <span class="usage-badge" :class="{ zero: !tag.usageCount }">
                {{ tag.usageCount }} KH
              </span>
            </div>

            <!-- Editable fields -->
            <div class="tag-edit-row">
              <input
                type="color"
                :value="tag.color"
                class="color-picker"
                title="Đổi màu"
                @change="patchTag(tag, { color: ($event.target as HTMLInputElement).value })"
              />
              <input
                type="text"
                :value="tag.emoji || ''"
                class="emoji-input"
                maxlength="4"
                placeholder="emoji"
                @blur="patchTag(tag, { emoji: ($event.target as HTMLInputElement).value || null })"
              />
              <input
                type="text"
                :value="tag.name"
                class="name-input"
                @blur="onRename(tag, ($event.target as HTMLInputElement).value)"
              />
            </div>

            <div class="tag-meta-row">
              <input
                type="text"
                :value="tag.category || ''"
                class="category-input"
                placeholder="Danh mục"
                @blur="patchTag(tag, { category: ($event.target as HTMLInputElement).value || null })"
              />
              <button class="icon-btn danger" title="Xoá tag" @click="confirmDelete(tag)">
                ✕
              </button>
            </div>

            <input
              v-if="editingDescId === tag.id"
              :value="tag.description || ''"
              class="desc-input"
              placeholder="Mô tả (mục đích sử dụng)..."
              @blur="onDescBlur(tag, ($event.target as HTMLInputElement).value)"
              @keydown.enter="($event.target as HTMLInputElement).blur()"
              ref="descInput"
            />
            <div v-else class="desc-preview" @click="editingDescId = tag.id">
              {{ tag.description || '+ thêm mô tả' }}
            </div>
          </div>
        </div>
      </section>
    </div>

    <!-- Create dialog -->
    <Teleport to="body">
      <div v-if="createDialog" class="dialog-backdrop" @click.self="createDialog = false">
        <div class="dialog">
          <h3>+ Thêm tag CRM mới</h3>
          <div class="dialog-form">
            <label>Tên *</label>
            <input v-model="newTag.name" type="text" placeholder="VD: VIP, Đầu tư, Hot lead..." />

            <label>Danh mục</label>
            <input v-model="newTag.category" type="text" placeholder="VD: Mức độ, Nguồn, Hành vi" list="categories-suggest" />
            <datalist id="categories-suggest">
              <option v-for="c in existingCategories" :key="c" :value="c" />
            </datalist>

            <label>Màu</label>
            <div class="color-row">
              <input v-model="newTag.color" type="color" />
              <div class="preset-colors">
                <button
                  v-for="c in PRESET_COLORS"
                  :key="c"
                  class="preset-color"
                  :style="`background:${c}`"
                  :class="{ selected: newTag.color === c }"
                  @click="newTag.color = c"
                />
              </div>
            </div>

            <label>Emoji (tuỳ chọn)</label>
            <input v-model="newTag.emoji" type="text" maxlength="4" placeholder="🔥 / ⭐ / 💎..." />

            <label>Mô tả</label>
            <textarea v-model="newTag.description" rows="2" placeholder="Mục đích, khi nào nên gắn..." />
          </div>
          <div class="dialog-actions">
            <button class="btn-link" @click="createDialog = false">Huỷ</button>
            <button class="btn-primary" :disabled="!newTag.name.trim() || creating" @click="submitCreate">
              {{ creating ? 'Đang tạo…' : 'Tạo' }}
            </button>
          </div>
        </div>
      </div>
    </Teleport>

    <!-- Delete confirm dialog -->
    <Teleport to="body">
      <div v-if="deleteTarget" class="dialog-backdrop" @click.self="deleteTarget = null">
        <div class="dialog">
          <h3>Xoá tag "{{ deleteTarget.name }}"?</h3>
          <p class="warn-text">
            Tag này đang gắn cho <strong>{{ deleteTarget.usageCount }}</strong> KH.
          </p>
          <label class="check-row">
            <input type="checkbox" v-model="removeFromContacts" />
            <span>Đồng thời gỡ tag khỏi {{ deleteTarget.usageCount }} KH ({{ removeFromContacts ? 'CÓ' : 'KHÔNG' }})</span>
          </label>
          <div class="dialog-actions">
            <button class="btn-link" @click="deleteTarget = null">Huỷ</button>
            <button class="btn-primary danger" @click="submitDelete">Xoá tag</button>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { api } from '@/api/index';
import { useToast } from '@/composables/use-toast';

interface CrmTag {
  id: string;
  name: string;
  color: string;
  emoji: string | null;
  description: string | null;
  category: string | null;
  order: number;
  isActive: boolean;
  usageCount: number;
}

const toast = useToast();

const PRESET_COLORS = [
  '#E53935', '#F4511E', '#FB8C00', '#FDD835', '#7CB342',
  '#43A047', '#00ACC1', '#1E88E5', '#3949AB', '#8E24AA',
  '#D81B60', '#5D4037', '#546E7A', '#90A4AE',
];

const tags = ref<CrmTag[]>([]);
const loading = ref(false);
const recounting = ref(false);
const filter = ref('');
const editingDescId = ref<string | null>(null);

const createDialog = ref(false);
const creating = ref(false);
const newTag = ref({ name: '', color: '#1E88E5', emoji: '', category: '', description: '' });

const deleteTarget = ref<CrmTag | null>(null);
const removeFromContacts = ref(true);

const filteredTags = computed(() => {
  if (!filter.value.trim()) return tags.value;
  const q = filter.value.toLowerCase();
  return tags.value.filter(t =>
    t.name.toLowerCase().includes(q) ||
    (t.category || '').toLowerCase().includes(q) ||
    (t.description || '').toLowerCase().includes(q),
  );
});

const groupedTags = computed(() => {
  const groups = new Map<string, CrmTag[]>();
  for (const t of filteredTags.value) {
    const cat = t.category || 'Chưa phân loại';
    if (!groups.has(cat)) groups.set(cat, []);
    groups.get(cat)!.push(t);
  }
  return [...groups.entries()].map(([category, list]) => ({ category, tags: list }));
});

const existingCategories = computed(() => {
  const s = new Set<string>();
  for (const t of tags.value) if (t.category) s.add(t.category);
  return [...s];
});

const activeCount = computed(() => tags.value.filter(t => t.usageCount > 0).length);
const unusedCount = computed(() => tags.value.filter(t => t.usageCount === 0).length);
const categoryCount = computed(() => existingCategories.value.length);

async function fetchTags(recount = false) {
  loading.value = true;
  try {
    const { data } = await api.get(`/crm-tags${recount ? '?recount=1' : ''}`);
    tags.value = data.tags || [];
  } catch (err) {
    console.error(err);
    toast.error('Không tải được tag CRM');
  } finally {
    loading.value = false;
  }
}

async function recount() {
  recounting.value = true;
  try {
    await fetchTags(true);
    toast.success('✓ Đã cập nhật số liệu');
  } finally {
    recounting.value = false;
  }
}

function openCreate() {
  newTag.value = { name: '', color: '#1E88E5', emoji: '', category: '', description: '' };
  createDialog.value = true;
}

async function submitCreate() {
  if (!newTag.value.name.trim()) return;
  creating.value = true;
  try {
    const { data } = await api.post('/crm-tags', {
      name: newTag.value.name.trim(),
      color: newTag.value.color,
      emoji: newTag.value.emoji || undefined,
      category: newTag.value.category || undefined,
      description: newTag.value.description || undefined,
    });
    tags.value = [...tags.value, data.tag];
    toast.success('✓ Đã tạo tag CRM');
    createDialog.value = false;
  } catch (err: any) {
    toast.error(err.response?.data?.error || 'Tạo tag thất bại');
  } finally {
    creating.value = false;
  }
}

async function patchTag(tag: CrmTag, body: Partial<CrmTag>) {
  // Optimistic
  const snapshot = { ...tag };
  Object.assign(tag, body);
  try {
    const { data } = await api.patch(`/crm-tags/${tag.id}`, body);
    Object.assign(tag, data.tag);
  } catch (err: any) {
    Object.assign(tag, snapshot);
    toast.error(err.response?.data?.error || 'Cập nhật thất bại');
  }
}

function onRename(tag: CrmTag, newName: string) {
  const t = newName.trim();
  if (!t || t === tag.name) return;
  patchTag(tag, { name: t });
}

function onDescBlur(tag: CrmTag, value: string) {
  editingDescId.value = null;
  const v = value.trim();
  if (v === (tag.description || '')) return;
  patchTag(tag, { description: v || null });
}

function confirmDelete(tag: CrmTag) {
  deleteTarget.value = tag;
  removeFromContacts.value = tag.usageCount > 0;
}

async function submitDelete() {
  if (!deleteTarget.value) return;
  try {
    await api.delete(`/crm-tags/${deleteTarget.value.id}`, {
      data: { removeFromContacts: removeFromContacts.value },
    });
    tags.value = tags.value.filter(t => t.id !== deleteTarget.value!.id);
    toast.success('✓ Đã xoá tag');
    deleteTarget.value = null;
  } catch (err: any) {
    toast.error(err.response?.data?.error || 'Xoá thất bại');
  }
}

onMounted(() => { void fetchTags(); });
</script>

<style scoped>
.crmtag-settings { max-width: 1200px; }
.settings-section-header { margin-bottom: 16px; }
.subtitle { font-size: 13px; color: var(--smax-grey-600); line-height: 1.5; margin: 0; }

/* ── Toolbar ──────────────────────────────────────────────────────────── */
.toolbar {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}
.toolbar-spacer { flex: 1; }
.toolbar-btn {
  background: var(--smax-grey-100);
  color: var(--smax-grey-700);
  border: 1px solid var(--smax-grey-200);
  border-radius: 7px;
  padding: 7px 14px;
  font-size: 12.5px;
  font-weight: 600;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 5px;
  transition: background 0.12s;
}
.toolbar-btn:hover:not(:disabled) { background: var(--smax-primary-soft); color: var(--smax-primary); }
.toolbar-btn:disabled { opacity: 0.5; cursor: not-allowed; }
.toolbar-btn.primary {
  background: var(--smax-primary);
  color: #fff;
  border-color: var(--smax-primary);
}
.toolbar-btn.primary:hover { background: #1e4cc7; color: #fff; }
.filter-input {
  border: 1px solid var(--smax-grey-200);
  border-radius: 7px;
  padding: 7px 11px;
  font-size: 13px;
  font-family: inherit;
  outline: none;
  min-width: 220px;
}
.filter-input:focus { border-color: var(--smax-primary); }

/* ── Stats ────────────────────────────────────────────────────────────── */
.stats-row {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  margin-bottom: 20px;
}
.stat-card {
  background: #fff;
  border: 1px solid var(--smax-grey-200);
  border-radius: 10px;
  padding: 12px 16px;
}
.stat-num {
  font-size: 26px;
  font-weight: 700;
  color: var(--smax-primary);
  line-height: 1.1;
}
.stat-label {
  font-size: 11px;
  color: var(--smax-grey-600);
  text-transform: uppercase;
  letter-spacing: 0.4px;
  margin-top: 2px;
}

/* ── Groups + Grid ────────────────────────────────────────────────────── */
.tag-group { margin-bottom: 20px; }
.group-title {
  font-size: 13px;
  font-weight: 700;
  color: var(--smax-grey-700);
  text-transform: uppercase;
  letter-spacing: 0.4px;
  margin: 0 0 10px;
  padding-bottom: 5px;
  border-bottom: 2px solid var(--smax-grey-100);
  display: flex;
  align-items: center;
  gap: 8px;
}
.group-count {
  background: var(--smax-grey-100);
  color: var(--smax-grey-700);
  padding: 1px 8px;
  border-radius: 9px;
  font-size: 11px;
  font-weight: 600;
}

.tag-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 10px;
}
.tag-card {
  background: #fff;
  border: 1px solid var(--smax-grey-200);
  border-radius: 10px;
  padding: 10px 12px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  transition: border-color 0.15s, box-shadow 0.15s;
}
.tag-card:hover {
  border-color: var(--smax-primary);
  box-shadow: 0 2px 8px rgba(33,150,243,0.1);
}
.tag-card.inactive { opacity: 0.5; }

.tag-preview-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}
.tag-preview {
  padding: 3px 11px;
  border-radius: 12px;
  font-size: 12.5px;
  font-weight: 600;
  border: 1.5px solid;
  display: inline-block;
}
.usage-badge {
  background: rgba(0, 200, 83, 0.12);
  color: #00897b;
  padding: 2px 9px;
  border-radius: 10px;
  font-size: 11px;
  font-weight: 700;
}
.usage-badge.zero {
  background: var(--smax-grey-100);
  color: var(--smax-grey-500);
}

.tag-edit-row {
  display: flex;
  align-items: center;
  gap: 6px;
}
.color-picker {
  width: 32px;
  height: 26px;
  border: 1px solid var(--smax-grey-200);
  border-radius: 5px;
  padding: 0;
  cursor: pointer;
}
.emoji-input, .name-input, .category-input {
  border: 1px solid transparent;
  border-radius: 5px;
  padding: 4px 7px;
  font-family: inherit;
  font-size: 13px;
  outline: none;
  background: transparent;
}
.emoji-input:hover, .name-input:hover, .category-input:hover { border-color: var(--smax-grey-200); }
.emoji-input:focus, .name-input:focus, .category-input:focus {
  border-color: var(--smax-primary);
  background: var(--smax-primary-soft);
}
.emoji-input { width: 50px; text-align: center; }
.name-input { flex: 1; font-weight: 600; }
.category-input { flex: 1; font-size: 12px; color: var(--smax-grey-600); }

.tag-meta-row {
  display: flex;
  align-items: center;
  gap: 6px;
}
.icon-btn {
  background: transparent;
  border: 1px solid transparent;
  width: 28px;
  height: 28px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  color: var(--smax-grey-500);
}
.icon-btn:hover { background: var(--smax-grey-100); }
.icon-btn.danger:hover { background: #ffebee; color: #c62828; border-color: #ef9a9a; }

.desc-preview {
  font-size: 12px;
  color: var(--smax-grey-500);
  cursor: pointer;
  padding: 4px 7px;
  border-radius: 5px;
  font-style: italic;
  min-height: 22px;
}
.desc-preview:hover {
  background: var(--smax-grey-50);
  color: var(--smax-grey-700);
}
.desc-input {
  width: 100%;
  border: 1.5px solid var(--smax-primary);
  border-radius: 5px;
  padding: 5px 8px;
  font-family: inherit;
  font-size: 12px;
  outline: none;
  background: var(--smax-primary-soft);
}

.loading-state, .empty-state {
  padding: 40px;
  text-align: center;
  color: var(--smax-grey-500);
}

/* ── Dialog ───────────────────────────────────────────────────────────── */
.dialog-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}
.dialog {
  background: #fff;
  border-radius: 12px;
  min-width: 420px;
  max-width: 480px;
  padding: 20px 24px;
  box-shadow: 0 12px 32px rgba(0,0,0,0.2);
}
.dialog h3 {
  margin: 0 0 14px;
  font-size: 16px;
}
.dialog-form {
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.dialog-form label {
  font-size: 11px;
  font-weight: 700;
  color: var(--smax-grey-600);
  text-transform: uppercase;
  letter-spacing: 0.4px;
  margin-top: 8px;
}
.dialog-form input[type="text"],
.dialog-form textarea {
  border: 1px solid var(--smax-grey-200);
  border-radius: 6px;
  padding: 7px 10px;
  font-family: inherit;
  font-size: 13px;
  outline: none;
}
.dialog-form input[type="text"]:focus,
.dialog-form textarea:focus { border-color: var(--smax-primary); }

.color-row {
  display: flex;
  align-items: center;
  gap: 10px;
}
.color-row input[type="color"] {
  width: 44px;
  height: 32px;
  border: 1px solid var(--smax-grey-200);
  border-radius: 6px;
  padding: 0;
  cursor: pointer;
}
.preset-colors {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  flex: 1;
}
.preset-color {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  border: 2px solid transparent;
  cursor: pointer;
  padding: 0;
}
.preset-color:hover { transform: scale(1.15); }
.preset-color.selected {
  border-color: #fff;
  box-shadow: 0 0 0 2px var(--smax-primary);
}

.dialog-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 16px;
  padding-top: 12px;
  border-top: 1px solid var(--smax-grey-100);
}
.btn-link {
  background: transparent;
  border: none;
  color: var(--smax-grey-600);
  font-size: 13px;
  cursor: pointer;
  padding: 6px 12px;
}
.btn-primary {
  background: var(--smax-primary);
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 7px 16px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
}
.btn-primary:disabled { opacity: 0.5; cursor: not-allowed; }
.btn-primary.danger { background: #c62828; }

.warn-text {
  color: var(--smax-grey-700);
  font-size: 13px;
  margin: 8px 0;
}
.check-row {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: var(--smax-grey-700);
  padding: 8px 10px;
  background: var(--smax-grey-50);
  border-radius: 6px;
  cursor: pointer;
}
</style>
