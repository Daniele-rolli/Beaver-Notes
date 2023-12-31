<!-- eslint-disable vue/multi-word-component-names -->
<template>
  <div class="general space-y-8 w-full max-w-xl">
    <section>
      <p class="mb-2">{{ translations.settings.apptheme || '-' }}</p>
      <div class="flex space-x-4 text-gray-600 dark:text-gray-200">
        <button
          v-for="item in themes"
          :key="item.name"
          :class="{
            'ring-2 ring-primary': theme.currentTheme.value === item.name,
          }"
          class="bg-input p-2 rounded-lg transition cursor-pointer"
          @click="theme.setTheme(item.name)"
        >
          <img :src="item.img" class="w-40 border-2 mb-1 rounded-lg" />
          <p class="capitalize text-center text-sm">
            {{ translations.settings[item.name] || item.name }}
          </p>
        </button>
      </div>
    </section>
    <section>
      <div>
        <p class="mb-2">{{ translations.settings.selectlanguage || '-' }}</p>
        <ui-select
          v-model="selectedLanguage"
          class="w-full"
          @change="updateLanguage"
        >
          <option
            v-for="language in languages"
            :key="language.code"
            :value="language.code"
          >
            {{ language.name }}
          </option>
        </ui-select>
      </div>
    </section>
    <section>
      <p class="mb-2">{{ translations.settings.selectfont || '-' }}</p>
      <ui-select
        id="fontSelect"
        v-model="selectedFont"
        class="w-full"
        @change="updateFont"
      >
        <option value="Arimo">Arimo</option>
        <option value="avenir">Avenir</option>
        <option value="EB Garamond">EB Garamond</option>
        <option value="'Helvetica Neue', sans-serif">Helvetica</option>
        <option value="open-dyslexic">Open Dyslexic</option>
        <option value="Ubuntu">Ubuntu</option>
      </ui-select>
    </section>
    <section>
      <p class="mb-2">{{ translations.settings.syncpath || '-' }}</p>
      <div class="flex items-center space-x-2">
        <ui-input
          v-model="state.dataDir"
          readonly
          :placeholder="translations.settings.pathplaceholder || '-'"
          class="w-full"
          @click="chooseDefaultPath"
        />
        <ui-button class="w-full" @click="chooseDefaultPath">
          {{ translations.settings.selectpath || '-' }}
        </ui-button>
      </div>
    </section>
    <section>
      <label class="flex items-center space-x-2">
        <input
          v-model="disableAppReminder"
          class="form-checkbox"
          type="checkbox"
          @change="updateDisableAppReminder"
        />
        <span class="inline-block align-middle">
          {{ translations.settings.syncreminder || '-' }}
        </span>
      </label>
    </section>
    <section>
      <p class="mb-2">{{ translations.settings.iedata || '-' }}</p>
      <div class="flex space-x-4">
        <div class="bg-input transition w-6/12 rounded-lg p-4">
          <div class="text-center mb-8 dark:text-gray-300 text-gray-600">
            <span
              class="
                p-5
                rounded-full
                bg-black
                dark:bg-white dark:bg-opacity-5
                bg-opacity-5
                inline-block
              "
            >
              <v-remixicon size="36" name="riFileUploadLine" />
            </span>
          </div>
          <ui-checkbox v-model="state.withPassword">
            {{ translations.settings.encryptwpasswd || '-' }}
          </ui-checkbox>
          <expand-transition>
            <ui-input
              v-if="state.withPassword"
              v-model="state.password"
              :placeholder="translations.settings.password || '-'"
              class="mt-2"
              style="-webkit-text-security: disc"
              autofocus
              @keyup.enter="exportData"
            />
          </expand-transition>
          <ui-button class="w-full mt-4" @click="exportData(defaultPath)">
            {{ translations.settings.exportdata || '-' }}</ui-button
          >
        </div>
        <div class="bg-input transition w-6/12 rounded-lg p-4 flex flex-col">
          <div class="text-center mb-6 dark:text-gray-300 text-gray-600">
            <span
              class="
                p-5
                rounded-full
                bg-black
                dark:bg-white dark:bg-opacity-5
                bg-opacity-5
                inline-block
              "
            >
              <v-remixicon size="36" name="riFileDownloadLine" />
            </span>
          </div>
          <div class="flex-grow"></div>
          <ui-button class="w-full mt-6" @click="importData(defaultPath)">
            {{ translations.settings.importdata || '-' }}
          </ui-button>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { shallowReactive, onMounted } from 'vue';
import { AES } from 'crypto-es/lib/aes';
import { Utf8 } from 'crypto-es/lib/core';
import { useTheme } from '@/composable/theme';
import { useStorage } from '@/composable/storage';
import { useDialog } from '@/composable/dialog';
import dayjs from '@/lib/dayjs';
import lightImg from '@/assets/images/light.png';
import darkImg from '@/assets/images/dark.png';
import systemImg from '@/assets/images/system.png';
import Mousetrap from '@/lib/mousetrap';
import enTranslations from './locales/en.json';
import itTranslations from './locales/it.json';
import deTranslations from './locales/de.json';
import zhTranslations from './locales/zh.json';
import '../../assets/css/passwd.css';

export const state = shallowReactive({
  dataDir: '',
});
export const dataDir = state.dataDir;

export default {
  setup() {
    const { ipcRenderer, path } = window.electron;
    const themes = [
      { name: 'light', img: lightImg },
      { name: 'dark', img: darkImg },
      { name: 'system', img: systemImg },
    ];

    const theme = useTheme();
    // eslint-disable-next-line no-unused-vars
    const dialog = useDialog();
    const storage = useStorage();

    const state = shallowReactive({
      dataDir: '',
      password: '',
      withPassword: false,
      lastUpdated: null,
    });

    let defaultPath = '';

    async function changeDataDir() {
      try {
        const {
          canceled,
          filePaths: [dir],
        } = await ipcRenderer.callMain('dialog:open', {
          title: 'Select directory',
          properties: ['openDirectory'],
        });

        if (canceled) return;

        showAlert(translations.settings.relaunch, {
          type: 'info',
          buttons: [translations.settings.relaunchbutton],
        });

        await storage.set('dataDir', dir);
        window.location.reload(); // Reload the page
      } catch (error) {
        console.error(error);
      }
    }

    function showAlert(message, options = {}) {
      ipcRenderer.callMain('dialog:message', {
        type: 'error',
        title: 'Alert',
        message,
        ...options,
      });
    }

    async function exportData() {
      try {
        const { canceled, filePaths } = await ipcRenderer.callMain(
          'dialog:open',
          {
            title: 'Export data',
            properties: ['openDirectory'],
          }
        );

        if (canceled) return;

        let data = await storage.store();
        data['default-path'] = defaultPath;
        data['lockedNotes'] = JSON.parse(localStorage.getItem('lockedNotes'));
        if (state.withPassword) {
          data = AES.encrypt(JSON.stringify(data), state.password).toString();
        }

        const folderName = dayjs().format('[Beaver Notes] YYYY-MM-DD');
        const folderPath = path.join(filePaths[0], folderName);
        const dataDir = await storage.get('dataDir', '', 'settings');

        await ipcRenderer.callMain('fs:ensureDir', folderPath);
        await ipcRenderer.callMain('fs:output-json', {
          path: path.join(folderPath, 'data.json'),
          data: { data },
        });
        await ipcRenderer.callMain('fs:copy', {
          path: path.join(dataDir, 'notes-assets'),
          dest: path.join(folderPath, 'assets'),
        });

        alert(`${translations.settings.exportmessage}"${folderName}"`);

        state.withPassword = false;
        state.password = '';
      } catch (error) {
        console.error(error);
      }
    }

    async function mergeImportedData(data) {
      try {
        const keys = [
          { key: 'notes', dfData: {} },
          { key: 'labels', dfData: [] },
        ];

        for (const { key, dfData } of keys) {
          const currentData = await storage.get(key, dfData);
          const importedData = data[key] ?? dfData;
          let mergedData;

          if (key === 'labels') {
            const mergedArr = [...currentData, ...importedData];

            mergedData = [...new Set(mergedArr)];
          } else {
            mergedData = { ...currentData, ...importedData };
          }

          await storage.set(key, mergedData);
        }

        window.location.reload();
      } catch (error) {
        console.error(error);
      }
    }
    async function importData() {
      try {
        const {
          canceled,
          filePaths: [dirPath],
        } = await ipcRenderer.callMain('dialog:open', {
          title: 'Import data',
          properties: ['openDirectory'],
        });

        if (canceled) return;

        let { data } = await ipcRenderer.callMain(
          'fs:read-json',
          path.join(dirPath, 'data.json')
        );

        if (!data) return showAlert(translations.settings.invaliddata);

        if (typeof data === 'string') {
          dialog.prompt({
            title: translations.settings.Inputpassword,
            body: translations.settings.body,
            okText: translations.settings.Import,
            cancelText: translations.settings.Cancel,
            placeholder: translations.settings.Password,
            onConfirm: (pass) => {
              try {
                const bytes = AES.decrypt(data, pass);
                const result = bytes.toString(Utf8);
                const resultObj = JSON.parse(result);

                mergeImportedData(resultObj);
              } catch (error) {
                showAlert(translations.settings.Invalidpassword);
                return false;
              }
            },
          });
        } else {
          mergeImportedData(data);
        }

        const dataDir = await storage.get('dataDir', '', 'settings');
        const importedDefaultPath = data['dataDir'];
        const importedLockedStatus = data['lockStatus'];
        const importedLockedNotes = data['lockedNotes'];
        localStorage.setItem('dataDir', importedDefaultPath);
        localStorage.setItem(
          'lockStatus',
          JSON.stringify(importedLockedStatus)
        );
        localStorage.setItem(
          'lockedNotes',
          JSON.stringify(importedLockedNotes)
        );

        await ipcRenderer.callMain('fs:copy', {
          // eslint-disable-next-line no-undef
          path: path.join(folderPath, 'assets'),
          dest: path.join(dataDir, 'notes-assets'),
        });
        window.reload();
      } catch (error) {
        console.error(error);
      }
    }

    async function chooseDefaultPath() {
      try {
        const {
          canceled,
          filePaths: [dir],
        } = await ipcRenderer.callMain('dialog:open', {
          title: 'Select directory',
          properties: ['openDirectory'],
        });

        if (canceled) return;

        showAlert(translations.settings.relaunch, {
          type: 'info',
          buttons: [translations.settings.relaunchbutton],
        });
        defaultPath = dir;
        localStorage.setItem('default-path', defaultPath);
        state.dataDir = defaultPath;
        window.location.reload(); // Reload the page
      } catch (error) {
        console.error(error);
      }
    }

    onMounted(() => {
      defaultPath = localStorage.getItem('default-path') || ''; // Set defaultPath here
      state.dataDir = defaultPath;
    });

    const shortcuts = {
      'mod+shift+u': importData,
      'mod+shift+e': exportData,
    };

    // Translations
    const translations = shallowReactive({
      settings: {
        apptheme: 'settings.apptheme',
        light: 'settings.light',
        dark: 'settings.dark',
        system: 'settings.system',
        selectlanguage: 'settings.selectlanguage',
        selectfont: 'settings.selectfont',
        syncpath: 'settings.syncpath',
        selectpath: 'settings.selectpath',
        iedata: 'settings.iedata',
        encryptwpasswd: 'settings.encryptwpasswd',
        exportdata: 'settings.exportdata',
        importdata: 'settings.importdata',
        pathplaceholder: 'settings.pathplaceholder',
        password: 'settings.password',
        Inputpassword: 'settings.Inputpassword',
        body: 'settings.body',
        Import: 'settings.Import',
        Cancel: 'settings.Cancel',
        Password: 'settings.password',
        Invalidpassword: 'settings.Invalidpassword',
        relaunch: 'settings.relaunch',
        relaunchbutton: 'settings.relaunchbutton',
        exportmessage: 'settings.exportmessage',
        invaliddata: 'settings.invaliddata',
        syncreminder: 'settings.syncreminder',
      },
    });

    onMounted(async () => {
      // Load translations
      const loadedTranslations = await loadTranslations();
      if (loadedTranslations) {
        Object.assign(translations, loadedTranslations);
      }
    });

    const loadTranslations = async () => {
      const selectedLanguage = localStorage.getItem('selectedLanguage') || 'en';
      try {
        const translationModule = await import(
          `../../pages/settings/locales/${selectedLanguage}.json`
        );
        return translationModule.default;
      } catch (error) {
        console.error('Error loading translations:', error);
        return null;
      }
    };

    Mousetrap.bind(Object.keys(shortcuts), (event, combo) => {
      shortcuts[combo]();
    });

    return {
      state,
      theme,
      themes,
      storage,
      translations,
      exportData,
      importData,
      changeDataDir,
      chooseDefaultPath,
      defaultPath,
    };
  },
  data() {
    return {
      disableAppReminder: localStorage.getItem('disableAppReminder') === 'true',
      selectedFont: localStorage.getItem('selected-font') || 'Arimo',
      selectedLanguage: localStorage.getItem('selectedLanguage') || 'en', // Initialize with a value from localStorage if available
      languages: [
        { code: 'en', name: 'English', translations: enTranslations },
        { code: 'it', name: 'Italiano', translations: itTranslations },
        { code: 'de', name: 'Deutsch', translations: deTranslations },
        { code: 'zh', name: '简体中文', translations: zhTranslations },
      ],
    };
  },
  mounted() {
    document.documentElement.style.setProperty(
      '--selected-font',
      this.selectedFont
    );
  },
  methods: {
    updateFont() {
      localStorage.setItem('selected-font', this.selectedFont);

      document.documentElement.style.setProperty(
        '--selected-font',
        this.selectedFont
      );
    },
    updateLanguage() {
      const languageCode = this.selectedLanguage;
      localStorage.setItem('selectedLanguage', languageCode);
      window.location.reload(); // Reload the page
    },
    updateDisableAppReminder() {
      localStorage.setItem(
        'disableAppReminder',
        this.disableAppReminder.toString()
      );
    },
  },
};
</script>
