---
title: "安卓语言切换流程"
date: "2018-03-26"
categories: 
    - "ANDROID"
---

# Settings

1. 在settings应用中，通过LocaleDragAndDropAdapter来实现语言列表拖拽切换语言的，在拖拽结束后会调用updateLocalesWhenAnimationStops表示当动画结束后，进行locale的更新操作

   ​

```
        public void updateLocalesWhenAnimationStops(final LocaleList localeList) {
        //在动画借宿后，更新locale设置
        if (localeList.equals(mLocalesToSetNext)) {
            return;
        }

        // This will only update the Settings application to make things feel more responsive,
        // the system will be updated later, when animation stopped.
        LocaleList.setDefault(localeList);

        mLocalesToSetNext = localeList;
        final RecyclerView.ItemAnimator itemAnimator = mParentView.getItemAnimator();
        itemAnimator.isRunning(new RecyclerView.ItemAnimator.ItemAnimatorFinishedListener() {
            @Override
            public void onAnimationsFinished() {
              
                if (mLocalesToSetNext == null || mLocalesToSetNext.equals(mLocalesSetLast)) {
                    // All animations finished, but the locale list did not change
                    return;
                }

                LocalePicker.updateLocales(mLocalesToSetNext); //当动画结束的时候，调用真正的更新
                mLocalesSetLast = mLocalesToSetNext;
                new CreateShortcut.ShortcutsUpdateTask(mContext).execute();

                mLocalesToSetNext = null;

                mNumberFormatter = NumberFormat.getNumberInstance(Locale.getDefault());
            }
        });
    }
```

2. LocalePicker.updateLocales(mLocalesToSetNext);当动画结束时，调用系统方法，更新locale，LocalePicker的位置：

`android\frameworks\base\core\java\com\android\internal\app\LocalePicker.java`

请求系统来更新系统的语言设置

```
     /**
     * Requests the system to update the system locale. Note that the system looks halted
     * for a while during the Locale migration, so the caller need to take care of it.
     *
     * @see #updateLocales(LocaleList)
     */
    public static void updateLocale(Locale locale) {
        updateLocales(new LocaleList(locale));
    }
    
    public static void updateLocales(LocaleList locales) {
        try {
            final IActivityManager am = ActivityManager.getService();
            final Configuration config = am.getConfiguration();

            config.setLocales(locales);
            config.userSetLocale = true;
            //1.am
            am.updatePersistentConfiguration(config);
            // Trigger the dirty bit for the Settings Provider.
            //2.backupmanager
            BackupManager.dataChanged("com.android.providers.settings");
        } catch (RemoteException e) {
            // Intentionally left blank
        }
    }
```

3. 通过`am.updatePersistentConfiguration(config);`来进行配置更新和持久化，am的真正实现为 ActivityManagerService.java，先确认权限，

```
    @Override
    public void updatePersistentConfiguration(Configuration values) {
        enforceCallingPermission(CHANGE_CONFIGURATION, "updatePersistentConfiguration()");
        enforceWriteSettingsPermission("updatePersistentConfiguration()");
        if (values == null) {
            throw new NullPointerException("Configuration must not be null");
        }

        int userId = UserHandle.getCallingUserId();

        synchronized(this) {
            updatePersistentConfigurationLocked(values, userId);
        }
    }

    private void updatePersistentConfigurationLocked(Configuration values, @UserIdInt int userId) {
        final long origId = Binder.clearCallingIdentity();
        try {
            updateConfigurationLocked(values, null, false, true, userId, false /* deferResume */);
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

4. 详细分析updateConfigurationLocked：

```
     /**
     * Do either or both things: (1) change the current configuration, and (2)
     * make sure the given activity is running with the (now) current
     * configuration.  Returns true if the activity has been left running, or
     * false if <var>starting</var> is being destroyed to match the new
     * configuration.
     *
     * @param userId is only used when persistent parameter is set to true to persist configuration
     *               for that particular user
     */
    private boolean updateConfigurationLocked(Configuration values, ActivityRecord starting,
            boolean initLocale, boolean persistent, int userId, boolean deferResume,
            UpdateConfigurationResult result) {
        int changes = 0;
        boolean kept = true;

        if (mWindowManager != null) {
            mWindowManager.deferSurfaceLayout();
        }
        try {
            if (values != null) {
                //1.1更新全局的配置信息
                changes = updateGlobalConfiguration(values, initLocale, persistent, userId,
                        deferResume);
            }

            kept = ensureConfigAndVisibilityAfterUpdate(starting, changes);
            //1.2当前的activity是否被开始destroy，以便重新创建
           
        } finally {
            if (mWindowManager != null) {
                mWindowManager.continueSurfaceLayout();
            }
        }

        if (result != null) {
            result.changes = changes;
            result.activityRelaunched = !kept;
        }
        return kept;
    }
```

5. updateGlobalConfiguration：更新默认的global配置，并通知各个监听器，比较长的代码，注释解读：该方法主要做了以下事情

```

    private int updateGlobalConfiguration(@NonNull Configuration values, boolean initLocale,
            boolean persistent, int userId, boolean deferResume) {
        mTempConfig.setTo(getGlobalConfiguration());
        final int changes = mTempConfig.updateFrom(values);
        
        if (changes == 0) {
            //如果没有变化，则return，return之前，由于 Activity.setRequestedOrientation 导致WindowManagerService.mWaitingForConfig被设置为true，从而使窗口被锁住，所以如果没有变化的话，performDisplayOverrideConfigUpdate来解冻窗口
            performDisplayOverrideConfigUpdate(values, deferResume, DEFAULT_DISPLAY);
            return 0;
        }
        .............
        EventLog.writeEvent(EventLogTags.CONFIGURATION_CHANGED, changes);

        if (!initLocale && !values.getLocales().isEmpty() && values.userSetLocale) {
            final LocaleList locales = values.getLocales();
            int bestLocaleIndex = 0;
            if (locales.size() > 1) {
                if (mSupportedSystemLocales == null) {
                    mSupportedSystemLocales = Resources.getSystem().getAssets().getLocales();
                }
                bestLocaleIndex = Math.max(0, locales.getFirstMatchIndex(mSupportedSystemLocales));
            }
            SystemProperties.set("persist.sys.locale",
                    locales.get(bestLocaleIndex).toLanguageTag());//设置保存
            LocaleList.setDefault(locales, bestLocaleIndex);
            
            
            mHandler.sendMessage(mHandler.obtainMessage(SEND_LOCALE_TO_MOUNT_DAEMON_MSG,
                    locales.get(bestLocaleIndex)));//1.1.1 发送SEND_LOCALE_TO_MOUNT_DAEMON_MSG消息，主要用于通知storageManager.setField(StorageManager.SYSTEM_LOCALE_KEY, l.toLanguageTag());存储管理，将系统语言更新
        }

        mConfigurationSeq = Math.max(++mConfigurationSeq, 1);
        mTempConfig.seq = mConfigurationSeq;

        //1.1.2 调用mStackSupervisor的回掉Update stored global config and notify everyone about the change.
        mStackSupervisor.onConfigurationChanged(mTempConfig);
       
        Slog.i(TAG, "Config changes=" + Integer.toHexString(changes) + " " + mTempConfig);
        //1.1.3 TODO(multi-display): Update UsageEvents#Event to include displayId.主要是event
        mUsageStatsService.reportConfigurationChange(mTempConfig,
                mUserController.getCurrentUserIdLocked());

        // TODO: If our config changes, should we auto dismiss any currently showing dialogs?
        mShowDialogs = shouldShowDialogs(mTempConfig);

        AttributeCache ac = AttributeCache.instance();
        if (ac != null) {
            ac.updateConfiguration(mTempConfig);
        }

        // Make sure all resources in our process are updated right now, so that anyone who is going
        // to retrieve resource values after we return will be sure to get the new ones. This is
        // especially important during boot, where the first config change needs to guarantee all
        //1.1.4 resources have that config before following boot code is executed.重要流程
        mSystemThread.applyConfigurationToResources(mTempConfig);

        //1.1.5 We need another copy of global config because we're scheduling some calls instead of
        // running them in place. We need to be sure that object we send will be handled unchanged.
        final Configuration configCopy = new Configuration(mTempConfig);
        if (persistent && Settings.System.hasInterestingConfigurationChanges(changes)) {
            Message msg = mHandler.obtainMessage(UPDATE_CONFIGURATION_MSG);
            msg.obj = configCopy;
            msg.arg1 = userId;
            mHandler.sendMessage(msg);
            //主要更新ContentResolver
        }

        for (int i = mLruProcesses.size() - 1; i >= 0; i--) {
            ProcessRecord app = mLruProcesses.get(i);
            try {
                if (app.thread != null) {
                    if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION, "Sending to proc "
                            + app.processName + " new config " + configCopy);
                    app.thread.scheduleConfigurationChanged(configCopy);
                    //重要流程，遍历
                }
            } catch (Exception e) {
            }
        }
        //ACTION_CONFIGURATION_CHANGED intent
        Intent intent = new Intent(Intent.ACTION_CONFIGURATION_CHANGED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY | Intent.FLAG_RECEIVER_REPLACE_PENDING
                | Intent.FLAG_RECEIVER_FOREGROUND
                | Intent.FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS);
        broadcastIntentLocked(null, null, intent, null, null, 0, null, null, null,
                AppOpsManager.OP_NONE, null, false, false, MY_PID, SYSTEM_UID,
                UserHandle.USER_ALL);
        if ((changes & ActivityInfo.CONFIG_LOCALE) != 0) {
             // ACTION_LOCALE_CHANGED intent
            intent = new Intent(Intent.ACTION_LOCALE_CHANGED);
            intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND
                    | Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND
                    | Intent.FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS);
            if (initLocale || !mProcessesReady) {
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
            }
            broadcastIntentLocked(null, null, intent, null, null, 0, null, null, null,
                    AppOpsManager.OP_NONE, null, false, false, MY_PID, SYSTEM_UID,
                    UserHandle.USER_ALL);
        }

        // Override configuration of the default display duplicates global config, so we need to
        // update it also. This will also notify WindowManager about changes.
        performDisplayOverrideConfigUpdate(mStackSupervisor.getConfiguration(), deferResume,
                DEFAULT_DISPLAY);

        return changes;
    }

```

以上方法主要做了以下几件事情：

- 设置persist.sys.locale，持久化保存

- 发送SEND_LOCALE_TO_MOUNT_DAEMON_MSG消息通知，storageManager

- 调用mStackSupervisor的回调mStackSupervisor.onConfigurationChanged(mTempConfig);

  ```
      /**
       * Notify that parent config changed and we need to update full configuration.
       * @see #mFullConfiguration
       */
      void onConfigurationChanged(Configuration newParentConfig) {
          mFullConfiguration.setTo(newParentConfig);
          mFullConfiguration.updateFrom(mOverrideConfiguration);
          for (int i = getChildCount() - 1; i >= 0; --i) {
              final ConfigurationContainer child = getChildAt(i);
              child.onConfigurationChanged(mFullConfiguration);
          }
      }
  ```

- mUsageStatsService.reportConfigurationChange(mTempConfig,mUserController.getCurrentUserIdLocked()); 报告UsageEvents

  ```
          @Override
          public void reportConfigurationChange(Configuration config, int userId) {
              if (config == null) {
                  Slog.w(TAG, "Configuration event reported with a null config");
                  return;
              }

              UsageEvents.Event event = new UsageEvents.Event();
              event.mPackage = "android";

              // This will later be converted to system time.
              event.mTimeStamp = SystemClock.elapsedRealtime();

              event.mEventType = UsageEvents.Event.CONFIGURATION_CHANGE;
              event.mConfiguration = new Configuration(config);
              mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
          }
  ```

- mSystemThread.applyConfigurationToResources(mTempConfig); 

  ActivityThread.Java

  ```
   public final void applyConfigurationToResources(Configuration config) {
          synchronized (mResourcesManager) {
              mResourcesManager.applyConfigurationToResourcesLocked(config, null);
          }
      }
  ```

  ResourcesManager.java

  ```
    public final boolean applyConfigurationToResourcesLocked(@NonNull Configuration config,
                                                               @Nullable CompatibilityInfo compat) {
          try {
              Trace.traceBegin(Trace.TRACE_TAG_RESOURCES,
                      "ResourcesManager#applyConfigurationToResourcesLocked");

              if (!mResConfiguration.isOtherSeqNewer(config) && compat == null) {
                  if (DEBUG || DEBUG_CONFIGURATION) Slog.v(TAG, "Skipping new config: curSeq="
                          + mResConfiguration.seq + ", newSeq=" + config.seq);
                  return false;
              }
              int changes = mResConfiguration.updateFrom(config);
              // Things might have changed in display manager, so clear the cached displays.
              mAdjustedDisplays.clear();

              DisplayMetrics defaultDisplayMetrics = getDisplayMetrics();

              if (compat != null && (mResCompatibilityInfo == null ||
                      !mResCompatibilityInfo.equals(compat))) {
                  mResCompatibilityInfo = compat;
                  changes |= ActivityInfo.CONFIG_SCREEN_LAYOUT
                          | ActivityInfo.CONFIG_SCREEN_SIZE
                          | ActivityInfo.CONFIG_SMALLEST_SCREEN_SIZE;
              }

              Resources.updateSystemConfiguration(config, defaultDisplayMetrics, compat);

              ApplicationPackageManager.configurationChanged();
              //Slog.i(TAG, "Configuration changed in " + currentPackageName());

              Configuration tmpConfig = null;

              for (int i = mResourceImpls.size() - 1; i >= 0; i--) {
                  ResourcesKey key = mResourceImpls.keyAt(i);
                  WeakReference<ResourcesImpl> weakImplRef = mResourceImpls.valueAt(i);
                  ResourcesImpl r = weakImplRef != null ? weakImplRef.get() : null;
                  if (r != null) {
                      if (DEBUG || DEBUG_CONFIGURATION) Slog.v(TAG, "Changing resources "
                              + r + " config to: " + config);
                      int displayId = key.mDisplayId;
                      boolean isDefaultDisplay = (displayId == Display.DEFAULT_DISPLAY);
                      DisplayMetrics dm = defaultDisplayMetrics;
                      final boolean hasOverrideConfiguration = key.hasOverrideConfiguration();
                      if (!isDefaultDisplay || hasOverrideConfiguration) {
                          if (tmpConfig == null) {
                              tmpConfig = new Configuration();
                          }
                          tmpConfig.setTo(config);

                          // Get new DisplayMetrics based on the DisplayAdjustments given
                          // to the ResourcesImpl. Update a copy if the CompatibilityInfo
                          // changed, because the ResourcesImpl object will handle the
                          // update internally.
                          DisplayAdjustments daj = r.getDisplayAdjustments();
                          if (compat != null) {
                              daj = new DisplayAdjustments(daj);
                              daj.setCompatibilityInfo(compat);
                          }
                          dm = getDisplayMetrics(displayId, daj);

                          if (!isDefaultDisplay) {
                              applyNonDefaultDisplayMetricsToConfiguration(dm, tmpConfig);
                          }

                          if (hasOverrideConfiguration) {
                              tmpConfig.updateFrom(key.mOverrideConfiguration);
                          }
                          r.updateConfiguration(tmpConfig, dm, compat);
                      } else {
                          r.updateConfiguration(config, dm, compat);
                      }
                      //Slog.i(TAG, "Updated app resources " + v.getKey()
                      //        + " " + r + ": " + r.getConfiguration());
                  } else {
                      //Slog.i(TAG, "Removing old resources " + v.getKey());
                      mResourceImpls.removeAt(i);
                  }
              }

              return changes != 0;
          } finally {
              Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
          }
      }
  ```

  Resources.updateSystemConfiguration(config, defaultDisplayMetrics, compat); 更新资源配置

  ```
  if ((configChanges & ActivityInfo.CONFIG_LOCALE) != 0) {
                      if (locales.size() > 1) {
                          // The LocaleList has changed. We must query the AssetManager's available
                          // Locales and figure out the best matching Locale in the new LocaleList.
                          String[] availableLocales = mAssets.getNonSystemLocales();
                          if (LocaleList.isPseudoLocalesOnly(availableLocales)) {
                              // No app defined locales, so grab the system locales.
                              availableLocales = mAssets.getLocales();
                              if (LocaleList.isPseudoLocalesOnly(availableLocales)) {
                                  availableLocales = null;
                              }
                          }

                          if (availableLocales != null) {
                              final Locale bestLocale = locales.getFirstMatchWithEnglishSupported(
                                      availableLocales);
                              if (bestLocale != null && bestLocale != locales.get(0)) {
                                  mConfiguration.setLocales(new LocaleList(bestLocale, locales));
                              }
                          }
                      }
                  }
  ```

  ApplicationPackageManager.configurationChanged();清空缓存

  ```
  static void configurationChanged() {
          synchronized (sSync) {
              sIconCache.clear();
              sStringCache.clear();
          }
      }
  ```

- app.thread.scheduleConfigurationChanged(configCopy);遍历每个activityThread，应用新的配置

  ActivityThread.java

  ```
          public void scheduleConfigurationChanged(Configuration config) {
              updatePendingConfiguration(config);
              sendMessage(H.CONFIGURATION_CHANGED, config);
          }
  ```

  CONFIGURATION_CHANGED:

  ```
           case CONFIGURATION_CHANGED:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "configChanged");
                      mCurDefaultDisplayDpi = ((Configuration)msg.obj).densityDpi;
                      mUpdatingSystemConfig = true;
                      try {
                          handleConfigurationChanged((Configuration) msg.obj, null);
                      } finally {
                          mUpdatingSystemConfig = false;
                      }
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
  ```

  ActivityThread.handleConfigurationChanged

  ```
  final void handleConfigurationChanged(Configuration config, CompatibilityInfo compat) {

          int configDiff = 0;

          // This flag tracks whether the new configuration is fundamentally equivalent to the
          // existing configuration. This is necessary to determine whether non-activity
          // callbacks should receive notice when the only changes are related to non-public fields.
          // We do not gate calling {@link #performActivityConfigurationChanged} based on this flag
          // as that method uses the same check on the activity config override as well.
          final boolean equivalent = config != null && mConfiguration != null
                  && (0 == mConfiguration.diffPublicOnly(config));

          synchronized (mResourcesManager) {
              if (mPendingConfiguration != null) {
                  if (!mPendingConfiguration.isOtherSeqNewer(config)) {
                      config = mPendingConfiguration;
                      mCurDefaultDisplayDpi = config.densityDpi;
                      updateDefaultDensity();
                  }
                  mPendingConfiguration = null;
              }

              if (config == null) {
                  return;
              }

              if (DEBUG_CONFIGURATION) Slog.v(TAG, "Handle configuration changed: "
                      + config);

              mResourcesManager.applyConfigurationToResourcesLocked(config, compat);
              updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                      mResourcesManager.getConfiguration().getLocales());

              if (mConfiguration == null) {
                  mConfiguration = new Configuration();
              }
              if (!mConfiguration.isOtherSeqNewer(config) && compat == null) {
                  return;
              }

              configDiff = mConfiguration.updateFrom(config);
              config = applyCompatConfiguration(mCurDefaultDisplayDpi);

              final Theme systemTheme = getSystemContext().getTheme();
              if ((systemTheme.getChangingConfigurations() & configDiff) != 0) {
                  systemTheme.rebase();
              }

              final Theme systemUiTheme = getSystemUiContext().getTheme();
              if ((systemUiTheme.getChangingConfigurations() & configDiff) != 0) {
                  systemUiTheme.rebase();
              }
          }

          ArrayList<ComponentCallbacks2> callbacks = collectComponentCallbacks(false, config);

          freeTextLayoutCachesIfNeeded(configDiff);

          if (callbacks != null) {
              final int N = callbacks.size();
              for (int i=0; i<N; i++) {
                  ComponentCallbacks2 cb = callbacks.get(i);
                  if (cb instanceof Activity) {
                      // If callback is an Activity - call corresponding method to consider override
                      // config and avoid onConfigurationChanged if it hasn't changed.
                      Activity a = (Activity) cb;
                      performConfigurationChangedForActivity(mActivities.get(a.getActivityToken()),
                              config);
                  } else if (!equivalent) {
                      performConfigurationChanged(cb, config);
                  }
              }
          }
      }

  ```



```
    
```





 app.thread.scheduleConfigurationChanged(configCopy);

回到

ActivityThread.java

```
        public void scheduleConfigurationChanged(Configuration config) {
            updatePendingConfiguration(config);
            sendMessage(H.CONFIGURATION_CHANGED, config);
        }
```

CONFIGURATION_CHANGED:

```

                case CONFIGURATION_CHANGED:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "configChanged");
                    mCurDefaultDisplayDpi = ((Configuration)msg.obj).densityDpi;
                    mUpdatingSystemConfig = true;
                    try {
                        handleConfigurationChanged((Configuration) msg.obj, null);
                    } finally {
                        mUpdatingSystemConfig = false;
                    }
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
```

ActivityThread.handleConfigurationChanged

```
 final void handleConfigurationChanged(Configuration config, CompatibilityInfo compat) {

        int configDiff = 0;

        // This flag tracks whether the new configuration is fundamentally equivalent to the
        // existing configuration. This is necessary to determine whether non-activity
        // callbacks should receive notice when the only changes are related to non-public fields.
        // We do not gate calling {@link #performActivityConfigurationChanged} based on this flag
        // as that method uses the same check on the activity config override as well.
        final boolean equivalent = config != null && mConfiguration != null
                && (0 == mConfiguration.diffPublicOnly(config));

        synchronized (mResourcesManager) {
            if (mPendingConfiguration != null) {
                if (!mPendingConfiguration.isOtherSeqNewer(config)) {
                    config = mPendingConfiguration;
                    mCurDefaultDisplayDpi = config.densityDpi;
                    updateDefaultDensity();
                }
                mPendingConfiguration = null;
            }

            if (config == null) {
                return;
            }

            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Handle configuration changed: "
                    + config);

            mResourcesManager.applyConfigurationToResourcesLocked(config, compat);
            updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                    mResourcesManager.getConfiguration().getLocales());

            if (mConfiguration == null) {
                mConfiguration = new Configuration();
            }
            if (!mConfiguration.isOtherSeqNewer(config) && compat == null) {
                return;
            }

            configDiff = mConfiguration.updateFrom(config);
            config = applyCompatConfiguration(mCurDefaultDisplayDpi);

            final Theme systemTheme = getSystemContext().getTheme();
            if ((systemTheme.getChangingConfigurations() & configDiff) != 0) {
                systemTheme.rebase();
            }

            final Theme systemUiTheme = getSystemUiContext().getTheme();
            if ((systemUiTheme.getChangingConfigurations() & configDiff) != 0) {
                systemUiTheme.rebase();
            }
        }

        ArrayList<ComponentCallbacks2> callbacks = collectComponentCallbacks(false, config);

        freeTextLayoutCachesIfNeeded(configDiff);

        if (callbacks != null) {
            final int N = callbacks.size();
            for (int i=0; i<N; i++) {
                ComponentCallbacks2 cb = callbacks.get(i);
                if (cb instanceof Activity) {
                    // If callback is an Activity - call corresponding method to consider override
                    // config and avoid onConfigurationChanged if it hasn't changed.
                    Activity a = (Activity) cb;
                    performConfigurationChangedForActivity(mActivities.get(a.getActivityToken()),
                            config);
                } else if (!equivalent) {
                    performConfigurationChanged(cb, config);
                }
            }
        }
    }

```



其中调用log打印

```
01-09 03:12:27.360: I/linlian(5221): LocaleDragAndDropAdapter.updateLocalesWhenAnimationStops()
01-09 03:12:27.420: I/linlian(5221): LocaleDragAndDropAdapter.onAnimationsFinished()
01-09 03:12:27.420: I/linlian(5221): LocaleDragAndDropAdapter.onAnimationsFinished()mLocalesToSetNext=[en_US,zh_CN_#Hans]
动画结束，准备更新locale
01-09 03:12:27.421: I/linlian(5221): LocalePicker.updateLocale()config={1.0 ?mcc?mnc [en_US,zh_CN_#Hans] ldltr sw320dp w320dp h509dp 240dpi nrml long port finger -keyb/v/h -nav/h appBounds=Rect(0, 0 - 480, 800) s.7}
01-09 03:12:27.422: I/linlian(714): ActivityManagerService.updateGlobalConfiguration changes=8196
01-09 03:12:27.422: I/linlian(714): ActivityManagerService.updateGlobalConfiguration values={1.0 ?mcc?mnc [en_US,zh_CN_#Hans] ldltr sw320dp w320dp h509dp 240dpi nrml long port finger -keyb/v/h -nav/h appBounds=Rect(0, 0 - 480, 800) s.7}
01-09 03:12:27.432: I/linlian(714): ActivityManagerService.updateGlobalConfiguration locales=[en_US,zh_CN_#Hans]
更新locale排序为 英语，简体中文
01-09 03:12:27.432: I/linlian(714): ActivityManagerService.updateGlobalConfiguration bestLocaleIndex=0
01-09 03:12:27.433: I/linlian(714): ActivityManagerService.updateGlobalConfiguration locales=Config changes=2004 {1.0 ?mcc?mnc [en_US,zh_CN_#Hans] ldltr sw320dp w320dp h509dp 240dpi nrml long port finger -keyb/v/h -nav/h appBounds=Rect(0, 0 - 480, 800) s.8}
遍历当前的ProcessRecord，开机的有 设置，短信，launcher，acore,latin输入法
01-09 03:12:27.457: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{9538ed4 5221:com.android.settings/1000}
01-09 03:12:27.457: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{a10987d 2720:com.android.mms/u0a17}
01-09 03:12:27.458: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{5353372 2324:com.android.launcher3/u0a16}
01-09 03:12:27.458: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{97fd3c3 5277:android.process.acore/u0a2}
01-09 03:12:27.458: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{e22e5a6 1686:com.android.inputmethod.latin/u0a50}
.........编译APP，会被多次调用 handleConfigurationChanged 会被多次调用
01-09 03:12:27.459: I/linlian(2720): ActivityThread.handleConfigurationChanged()
01-09 03:12:27.459: I/linlian(2720): java.lang.RuntimeException
01-09 03:12:27.459: I/linlian(2720): 	at android.app.ActivityThread.handleConfigurationChanged(ActivityThread.java:4972)
01-09 03:12:27.459: I/linlian(2720): 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1705)
01-09 03:12:27.459: I/linlian(2720): 	at android.os.Handler.dispatchMessage(Handler.java:106)
01-09 03:12:27.459: I/linlian(2720): 	at android.os.Looper.loop(Looper.java:164)
01-09 03:12:27.459: I/linlian(2720): 	at android.app.ActivityThread.main(ActivityThread.java:6503)
01-09 03:12:27.459: I/linlian(2720): 	at java.lang.reflect.Method.invoke(Native Method)
01-09 03:12:27.459: I/linlian(2720): 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
01-09 03:12:27.459: I/linlian(2720): 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)
01-09 03:12:27.459: I/linlian(2324): ActivityThread.handleConfigurationChanged()
log穿插着ProcessRecord
01-09 03:12:27.471: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{351e46c 2749:com.android.onetimeinitializer/u0a19}
01-09 03:12:27.471: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{4ff5a35 2396:com.android.managedprovisioning/u0a15}
01-09 03:12:27.472: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{833beca 2520:com.android.exchange/u0a48}
01-09 03:12:27.472: I/linlian(714): ActivityManagerService.ProcessRecord app=ProcessRecord{c65f83b 2702:com.android.gallery3d/u0a49}

当前的应用
01-09 03:12:27.559: I/linlian(5221): ActivityThread.handleConfigurationChanged()Activity a=com.android.settings.SubSettings@e58e930

```

































