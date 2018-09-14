## InsertDialog

### 拉去数据的时机

#### 1，当收到灭屏广播的时候。  

#### 2，主进程初始化的时候。

### 请求过程

#### 1，isStartPageEnabled()  云控string_insert_section 返回为true 和 （配置文件不存在 或者 hasCloudConfigUpdate514()返回为false)，才去请求配置文件。

#### 2，启动页总接口。

```
//启动页配置文件请求
public static void requestInsertConfigData(final Context context) {
    InsertJsonRequest jsonRequest = new InsertJsonRequest(getInsertDataRequestUrl(context), new Response.Listener<JSONObject>() {
        @Override
        public void onResponse(JSONObject response) {
            if (response != null) {
                LauncherLog.releaseLog(TAG, "request insert config data succeed.");
                InsertDataParseHelper.saveInsertConfigData(context, response, true);
                CommonPreference.getInstance().setConfigUpdateIn514(true);
            }
        }
    }, new Response.ErrorListener() {
        @Override
        public void onErrorResponse(VolleyError error) {
            LauncherLog.releaseLog(TAG, "request insert config data failed. error = " + error.getMessage());
            UserBehaviorIPCManager.getInstance().report(false, InfocConstants.LAUNCHER_STARTPAGE_CONFIG,
                    "status", "3",
                    "showtime", KOtherMessageHelper.getStringDate(System.currentTimeMillis()));
            try {
                JSONObject configData = new JSONObject(DEFAULT_CONFIG_FILE);
                InsertDataParseHelper.saveInsertConfigData(context, configData, false);
            } catch (JSONException e) {
                e.printStackTrace();
            }
            UserBehaviorIPCManager.getInstance().report(false, "launcher_sp_error",
                    "reason_1", "1",
                    "reason_2", (error == null ? "no reason" : error.getMessage()));
        }
    });
    jsonRequest.setShouldCache(false);
    jsonRequest.setRetryPolicy(new DefaultRetryPolicy(REQUEST_TIMEOUT_MS,
            DefaultRetryPolicy.DEFAULT_MAX_RETRIES,
            DefaultRetryPolicy.DEFAULT_BACKOFF_MULT));
    getRequestQueue(context).add(jsonRequest);
}
```

```
// 启动页总接口url
private static String getInsertDataRequestUrl(Context context) {
    StringBuilder builder = new StringBuilder();
    builder.append(INSERT_BASE_URL);
    builder.append(TYPE_INTERFACE);
    builder.append("mcc=");
    builder.append(Commons.getMcc());
    builder.append("&apkver=");
    builder.append(KPackageManager.getPackageVersion(context, context.getPackageName()));
    builder.append("&aid=");
    builder.append(getInsertXaid());
    final String url = builder.toString();
    LauncherLog.releaseLog(TAG, " url = " + url);
    return url;
}
```

#### 3，配置文件请求成功，保存配置文件。

```
//保存启动页配置文件到本地
public static void saveInsertConfigData(final Context context, final JSONObject configData, final boolean needReport) {
    if (configData == null) {
        return;
    }
    LauncherLog.releaseLog(TAG, "save insert config data. " + configData.toString());
    ThreadManager.post(ThreadManager.THREAD_BACKGROUND, new Runnable() {
        @Override
        public void run() {
            Map<String, List<InsertConfigBean>> configMap = new HashMap<>();
            try {
                final String updateTime = configData.optString(KEY_CONFIG_UPDATE_TIME);
                saveInsertConfigUpdateTime(updateTime);
                final String nextUrl = configData.optString(KEY_CONFIG_NEXT_URL);
                saveInsertConfigNextUrl(nextUrl);

                //解析启动页配置信息
                final JSONArray insertList = configData.getJSONArray(KEY_CONFIG_LIST);
                List<InsertConfigBean> insertConfigList = parseConfigBean(insertList);
                if (insertConfigList.size() > 0) {
                    configMap.put(KEY_INSERT_CONFIG, insertConfigList);
                }
                LauncherLog.releaseLog(TAG, "saveInsertConfigData insertConfigList.size="
                        + insertConfigList.size());

                //liujia v5.29 add 解析小豹关怀页配置信息
                final JSONArray cheetahList = configData.getJSONArray(KEY_CHEETCARE_CONFIG_LIST);
                List<InsertConfigBean> cheetConfigList = parseConfigBean(cheetahList);
                if (cheetConfigList.size() > 0) {
                    configMap.put(KEY_CHEETCARE_CONFIG, cheetConfigList);
                }
                LauncherLog.releaseLog(TAG, "saveInsertConfigData cheetConfigList.size="
                        + cheetConfigList.size());

                saveConfigToFile(context, configMap, needReport);
            } catch (Exception e) {
                e.printStackTrace();
                LauncherLog.releaseLog(TAG, "saveInsertConfigData failed. error = " + e.getMessage());
            }
        }
    });

}
```

#### 解析json文件

```
/**
 * 解析配置信息
 * 每个类型的配置  返回一个LIST
 * @param configList
 * @return
 */
private static List<InsertConfigBean> parseConfigBean(final JSONArray configList) {
    List<InsertConfigBean> resultList = new ArrayList<>();
    try {
        if (configList != null && configList.length() > 0) {
            final int length = configList.length();
            for (int i = 0; i < length; i++) {
                final JSONObject object = (JSONObject) configList.get(i);
                final int id = object.optInt(KEY_CONFIG_ID);
                final int pid = object.optInt(KEY_CONFIG_PID);
                final int cid = object.optInt(KEY_CONFIG_CID);
                int st_time = object.optInt(KEY_CONFIG_ST_TIME);
                if (st_time == 24) {
                    st_time = 0;
                }
                int end_time = object.optInt(KEY_CONFIG_END_TIME);
                if (end_time == 0) {
                    end_time = 24;
                }
                final String begin_date = object.optString(KEY_CONFIG_BEGIN_DATE);
                final String expire_date = object.optString(KEY_CONFIG_EXPIRE_DATE);
                long beginDate = parseDate(begin_date);
                long expireDate = parseDate(expire_date);
                InsertConfigBean.Builder builder = new InsertConfigBean.Builder();
                builder.setId(id)
                        .setClassId(pid)
                        .setSubClassId(cid)
                        .setStartTime(st_time)
                        .setEndTime(end_time)
                        .setBeginDate(beginDate)
                        .setExpireDate(expireDate == 0 ? Long.MAX_VALUE : expireDate);
                final InsertConfigBean build = builder.build();
                resultList.add(build);
            }
            Collections.sort(resultList, new InsertDataComparator());
            return resultList;
        }
    } catch (Exception e) {

    }
    return resultList;
}
```

```
private static long parseDate(String dateStr) {
        if ("".equals(dateStr)) {
            return 0;
        } else {
//            Locale locale = LauncherApplication.getLauncherApplication().getResources().getConfiguration().locale;
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd", Locale.getDefault());
            Date date;
            try {
                date = sdf.parse(dateStr);
                return milliseToDays(date.getTime());
            } catch (ParseException e) {
                e.printStackTrace();
            }
            return 0;
        }
    }
```

#### 将配置信息保存到本地文件

```
private static void saveConfigToFile(Context context,
                                     Map<String, List<InsertConfigBean>> items,
                                     final boolean needReport) {
    final String filePath = getSaveInsertConfigFilePath(context);
    FileOutputStream fos = null;
    ObjectOutputStream oos = null;
    try {
        fos = new FileOutputStream(filePath);
        oos = new ObjectOutputStream(fos);
        oos.writeObject(items);
        updateInsertConfig = true;
        if (needReport) {
            UserBehaviorIPCManager.getInstance().report(false, InfocConstants.LAUNCHER_STARTPAGE_CONFIG,
                    "status", status + "",
                    "showtime", KOtherMessageHelper.getStringDate(System.currentTimeMillis()));
            status = 2;
        }
        LauncherLog.releaseLog(TAG, "saveInsertConfigToFile successfully");
    } catch (IOException e) {
        e.printStackTrace();
        LauncherLog.releaseLog(TAG, "saveInsertConfigToFile failed " + e.toString());
    } finally {
        try {
            if (fos != null) {
                fos.close();
            }
            if (oos != null) {
                oos.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 展示时机

#### 1，收到NotificationService.TYPE_USER_PRESENT广播的时候调用。

#### 2，在Launcher  onResume时云控launcher_startpage_ifshow_othersapp返回false。

### 执行showOtherMessage

#### 1，(hasSplash) ? true(go on) : false(return)。  判断是否执行了引导界面。

```
if (!hasSplashed) {
    return;
}
```

#### 2，(isReceiveUnLock) ?  true(go on) : false(return)。  判断是否是在解锁后界面。

```
if(!isReceiveUnLock) {
    return;
}
```

#### 3，(isShowMessageOnOtherApp) ? true(go on) : false[(mPaused || !mScreenOffOrLockPaused ) ? true(return) : false(go on )]。    判断云控值，和activity的状态。     

```
if (!isShowMessageOnOtherApp()) {
    if (mPaused || !mScreenOffOrLockPaused) {
        InterstitialAdLoader.getInstance().reportPageTypeData(11);
        LauncherLog.releaseLog(TAG, "launcher is paused or mScreenOffOrLockPaused is  " +
                mScreenOffOrLockPaused);
        return;
    }
}
```

#### 4，(isFinishing) ? true(go on ) : false (showInsertDialog()方法)    判断activity是否处于活跃状态(false)还是处于等待回收状态(true)。

```
if (!isFinishing()) {
    LauncherLog.releaseLog(TAG1, "start to get bean for show");
    final InsertDataBean insertDataBean = KOtherMessageHelper.getValidInsertBeanForLauncher(Launcher.this, InterstitialAdLoader.getInstance().isAdReady());
    //判断广告是否准好
    if (insertDataBean != null) {
        LauncherLog.releaseLog(TAG1, "start show insertDataBean.getType---" + insertDataBean.getType() + "---" + insertDataBean.getStartTime());
        showInsertDialog(insertDataBean, InsertDialog.SOURCE_LAUNCHER);
    } else {
        LauncherLog.releaseLog(TAG1, "bean is null, not show");
    }
}
```

#### 5，InterstitialAdLoader.getInstance().isAdReady()，判断广告是否准备好。

```
InterstitialAdLoader.getInstance().isAdReady()
```

#### 6，SPRuntimeCheck.isMainLauncherProcess() ？true (go on) : false (throw new IllegalStateException())。

```
if (!SPRuntimeCheck.isMainLauncherProcess()) {
    throw new IllegalStateException("only run in main launcher process!");
}
```

#### 7，SettingsModel.getInstance().getUnlockShowPage() ？ true(go on) : false (return)    设置页问候语按钮是否打开(默认云控打开)。

```
if (!SettingsModel.getInstance().getUnlockShowPage()) {
    LauncherLog.releaseLog(TAG, " not enabled by setting!");
    InterstitialAdLoader.getInstance().reportPageTypeData(1);
    return null;
}
```

#### 8，isStartPageEnabled()？true(go on) : false(return)   云控string_insert_section 返回为true。

```
if (!isStartPageEnabled()) {
    InterstitialAdLoader.getInstance().reportPageTypeData(2);
    LauncherLog.releaseLog(TAG, "locker enabled.");
    return null;
}
```

#### 9，isOtherDay() ? true(清空一些insert信息) :false(go on)  判断是否在同一天，同一天继续执行，不在同一天重设所有数值。

```
if (isOtherDay()) {
    resetInsertInfo();
}
```

#### 10，isStartPageShowExceedTimes() ？ true(return ) : false(go on)。是否超过最大展示次数。

```
if (isStartPageShowExceedTimes()) {
    InterstitialAdLoader.getInstance().reportPageTypeData(10);
    LauncherLog.releaseLog(TAG, "start page show  exceed max times");
    return null;
}
```

#### 11，获取配置文件。判断是否为null ， 为null return 。

```
mInsertConfigListFromFile = InsertDataParseHelper.getInsertConfigFromFile(context);
if (mInsertConfigListFromFile == null || mInsertConfigListFromFile.size() == 0) {
    LauncherLog.releaseLog(TAG, " no config data!");
    InterstitialAdLoader.getInstance().reportPageTypeData(5);
    return null;
}
if (insertDataBean != null && (insertDataBean.getEndTime() <= getCurrHour() || insertDataBean.getDay() != getCurrDay())) {
    insertDataBean = null;
    InterstitialAdLoader.getInstance().reportPageTypeData(3);
    LauncherLog.releaseLog(TAG, " insertData bean set null ");
}
```

#### 12，获取下一条即将展示的InsertConfigBean,预拉取使用。

```
//获取下一个即将展示的InsertConfigBean, 预拉取用
public static InsertConfigBean getNextValidInsertBean(List<InsertConfigBean> insertDataList) {
    return getNextValidInsertBean(insertDataList, false);
}

public static InsertConfigBean getNextValidInsertBean(List<InsertConfigBean> insertDataList, boolean skipFail) {
    final int currHour = getCurrHour();
    if (insertDataList != null && insertDataList.size() > 0) {
        final int size = insertDataList.size();
        for (int i = 0; i < size; i++) {
            final InsertConfigBean insertConfigBean = insertDataList.get(i);
            if (skipFail && isInRetryList(insertConfigBean)) {
                continue;
            }
            long days = InsertDataParseHelper.milliseToDays(System.currentTimeMillis());
            if (insertConfigBean.beginDate > days || insertConfigBean.expireDate < days) {
                LauncherLog.releaseLog(TAG, "I am not taken into effect... No Hurry :) " +
                        "---beginDate---" + insertConfigBean.beginDate + "---" + insertConfigBean.expireDate);
                continue;
            }
            if (insertConfigBean.startTime - 1 <= currHour && currHour < insertConfigBean.endTime
                    && checkInsertBeanValid(insertConfigBean)) {
                return insertConfigBean;
            }
        }
    }
    return null;
}
```

#### 13，是否预拉取下一个可展示的bean对象

##### 1，下一个可展示的配置bean不为空

##### 2，上一次展示的数据bean为空或者上一次展示。

```
if (nextValidInsertConfigBean != null
        && (insertDataBean == null
        || !isSameDataBean(nextValidInsertConfigBean, insertDataBean))) {
    LauncherLog.releaseLog(TAG, " nextValidInsertConfigBean =  " + nextValidInsertConfigBean.toString() + ((insertDataBean == null) ? " insertDataBean is null" : insertDataBean.toString()));

    InsertDataRequestHelper.requestInsertSubData(context, nextValidInsertConfigBean.id, nextValidInsertConfigBean.classId, new InsertDataRequestHelper.InsertRequestListener() {
        @Override
        public void onRequestSucceed(JSONObject object) {
            LauncherLog.releaseLog(TAG, "  request insert data bean successfully. ");
            insertDataBean = InsertDataParseHelper.parseSubInsertData(object,
                    getShouldDisplayInsertType(nextValidInsertConfigBean),
                    nextValidInsertConfigBean.startTime, nextValidInsertConfigBean.endTime,
                    nextValidInsertConfigBean.beginDate, nextValidInsertConfigBean.expireDate);
            preloadNextInsertImage(context, insertDataBean);
        }

        @Override
        public void onRequestFailed(String e) {
            LauncherLog.releaseLog(TAG, " request insert data bean failed. ");
        }
    });
} else {
    LauncherLog.releaseLog(TAG, "nextValidInsertConfigBean is null  or  has preloaded  do not request insert data bean. ");
}
```

#### 14，分接口数据请求。

```
public static void requestInsertSubData(final Context context, final int id, final int pid, final InsertRequestListener listener) {
    InsertJsonRequest jsonRequest = new InsertJsonRequest(getInsertDataSubRequestUrl(context, id, pid), new Response.Listener<JSONObject>() {
        @Override
        public void onResponse(JSONObject response) {
            LauncherLog.releaseLog(TAG, "request insert sub data succeed. data = " + response.toString());
            if (listener != null) {
                listener.onRequestSucceed(response);
            }
            if (response != null) {
                try {
                    final JSONArray insertList = response.getJSONArray(KEY_SUB_INSERT_CONFIG);
                    final JSONArray cheetCareList = response.getJSONArray(KEY_SUB_CHEETCARE_CONFIG);
                    //服务端判断updateTime过时，重新下发config文件
                    //liujia v5.29 add 如果启动页或者小豹关怀任一配置文件更新，均需要更新本地的配置文件
                    if (insertList.length() > 0 || cheetCareList.length() > 0) {
                        InsertDataParseHelper.saveInsertConfigData(context, response, false);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }, new Response.ErrorListener() {
        @Override
        public void onErrorResponse(VolleyError error) {
            LauncherLog.releaseLog(TAG, "request insert sub data failed. error = " + error.getMessage());
            if (listener != null) {
                listener.onRequestFailed(error.getMessage());
            }
        }
    });
    jsonRequest.setShouldCache(false);
    jsonRequest.setRetryPolicy(new DefaultRetryPolicy(REQUEST_TIMEOUT_MS,
            DefaultRetryPolicy.DEFAULT_MAX_RETRIES,
            DefaultRetryPolicy.DEFAULT_BACKOFF_MULT));
    getRequestQueue(context).add(jsonRequest);
}
```

#### 15，启动页分接口url拼接

```
private static String getInsertDataSubRequestUrl(Context context, int id, int pid) {
        StringBuilder builder = new StringBuilder();
        //TODO use next Url from config file?
//        builder.append(getSubInsertBaseUrl());
        builder.append(INSERT_BASE_URL);
        builder.append(TYPE_SUB_INTERFACE);
        builder.append("sc_id=");//对应InsertBean中的id
        builder.append(id);
        builder.append("&pid=");//对应InsertBean中的pid
        builder.append(pid);
        builder.append("&updateTime=");
        builder.append(getInsertConfigUpdateTime());
        builder.append("&mcc=");
        builder.append(Commons.getMcc());
        builder.append("&apkver=");
        builder.append(KPackageManager.getPackageVersion(context, context.getPackageName()));
        builder.append("&aid=");
        builder.append(getInsertXaid());
        builder.append("&width=");
        builder.append(DimenUtils.getWindowWidth(context));
        builder.append("&height=");
        builder.append(DimenUtils.getWindowHeight(context));
        final String subUrl = builder.toString();
        LauncherLog.releaseLog(TAG, " subUrl = " + subUrl);
        return subUrl;
    }
```

#### 16，数据解析。

```
public static InsertDataBean parseSubInsertData(JSONObject object, int type, int startTime,
                                                int endTime, long beginDate, long expireDate) {
    if (object == null) {
        return null;
    }
    InsertDataBean dataBean = new InsertDataBean();
    dataBean.setType(type);
    dataBean.setStartTime(startTime);
    dataBean.setEndTime(endTime);
    dataBean.setBeginDate(beginDate);
    dataBean.setExpireDate(expireDate);
    final String coverUrl1 = object.optString(KEY_SUB_COVER_URL);
    dataBean.setCoverUrl(coverUrl1);
    final String thumbUrl1 = object.optString(KEY_SUB_THUMB_URL);
    dataBean.setThumbUrl(thumbUrl1);
    dataBean.setDay(KOtherMessageHelper.getCurrDay());
    return dataBean;
}
```

#### 17，应该获取的弹窗类型。

```
public static int getShouldDisplayInsertType(InsertConfigBean insertConfigBean) {
    if (insertConfigBean == null) {
        return -1;
    }
    final int classId = insertConfigBean.classId;
    final int subClassId = insertConfigBean.subClassId;
    switch (classId) {
        case 1:
            return subClassId == 5 ? KMessageUtils.MESSAGE_WEATHER_TODAY : KMessageUtils.MESSAGE_WEATHER_FORECAST;
        case 2:
            if (subClassId == 7) {
                return KMessageUtils.MESSAGE_GREETING_MORNING;
            } else if (subClassId == 8) {
                return KMessageUtils.MESSAGE_GREETING_NOON;
            } else if (subClassId == 9) {
                return KMessageUtils.MESSAGE_GREETING_NIGHT;
            }
        case 3:
            return KMessageUtils.MESSAGE_DAILY_WALLPAPER;
        case 4:
            return KMessageUtils.MESSAGE_CUSTOM_THEME;
        case 15:
            return subClassId;
        default:
            return -1;
    }
}
```

#### 18，加载图片。

```
preloadNextInsertImage(context, insertDataBean);
```

```
private static void preloadNextInsertImage(Context context, InsertDataBean bean) {
        if (context == null || bean == null) {
            return;
        }
//        final boolean isWallPaper = bean.getType() == KMessageUtils.MESSAGE_DAILY_WALLPAPER;
//        final String imageUrl = isWallPaper ? bean.coverUrl : bean.thumbUrl;
        final String imageUrl = bean.thumbUrl;
        if (!hasPreloadSucceed(context, imageUrl)) {
            preloadNextInsertImage(context, imageUrl, bean);
        }

    }
```

```
private static void preloadNextInsertImage(final Context context, String url, final InsertDataBean bean) {
    File picFile = VolleyInstance.getInstance(context).getCacheFile(url);
    if (null != picFile && picFile.exists()) {
        return;
    }
    LauncherLog.releaseLog(TAG, " preloadNextInsertImage  url---" + url);
    VolleyInstance.getInstance(context).downloadFile(url, Long.MAX_VALUE, new VolleyInstance.DownloadListener() {
        @Override
        public void onSaved(long size) {

        }

        @Override
        public void onFailed(Throwable cause) {
            UserBehaviorIPCManager.getInstance().report(false, "launcher_sp_error",
                    "reason_1", "2",
                    "reason_2", (cause == null ? "no reason" : cause.getMessage()));

            // 保存下一个config bean
            if (bean.type == KMessageUtils.MESSAGE_CUSTOM_THEME ||
                    bean.type == KMessageUtils.MESSAGE_DAILY_WALLPAPER) {
                if (NetworkUtil.IsNetworkAvailable(context)) {
                    countRetryTimes(bean);
                    getNextValidInsertBean(mInsertConfigListFromFile, true);
                }
            }
        }
    });
}
```

#### 19，判断是否是有效数据。

```
private static boolean isValidInsertDataBean(InsertDataBean insertDataBean) {
    if (insertDataBean == null) {
        LauncherLog.releaseLog(TAG, "isValidInsertDataBean false   insertDataBean == null");
        return false;
    }
    //加入逻辑： 如果不在限定的日期内，则return false
    long days = InsertDataParseHelper.milliseToDays(System.currentTimeMillis());
    if (insertDataBean.beginDate > days || insertDataBean.expireDate < days) {
        LauncherLog.releaseLog(TAG, "isValidInsertDataBean false   not in Date interval." +
                "---beginDate---" + insertDataBean.getBeginDate() + "---" + insertDataBean.getExpireDate());
        return false;
    }
    final int currHour = getCurrHour();
    if (insertDataBean.startTime > currHour || currHour >= insertDataBean.endTime) {
        LauncherLog.releaseLog(TAG, "isValidInsertDataBean false   not in valid period." + "---getStartTime---" + insertDataBean.getStartTime() + "---" + insertDataBean.getEndTime());
        return false;
    }

    //如果包含就不展示 因为之前已经展示过了
    if (!checkInsertBeanValid(insertDataBean)) {
        LauncherLog.releaseLog(TAG, "isValidInsertDataBean false    has show in this period." + "---getStartTime---" + insertDataBean.getStartTime() + "---" + insertDataBean.getEndTime());
        return false;
    }
    LauncherLog.releaseLog(TAG, " isValidInsertDataBean true---insertDataBean" + "---getStartTime---" + insertDataBean.getStartTime() + "---" + insertDataBean.getEndTime());

    return true;
}
```

#### 20，是否达到间隔时间。

```
if (!isTimeIntervalIllegal()) {
    LauncherLog.releaseLog(TAG, " time interval not illegal.");
    InterstitialAdLoader.getInstance().reportPageTypeData(6);
    return null;
}
```

```
private static boolean isTimeIntervalIllegal() {
    long lastShowTime = 0;
    if (SPRuntimeCheck.isMainLauncherProcess()) {
        lastShowTime = CommonPreference.getInstance().getLastInsertShowTime();
    }
    long interval = getStartpageInterval();
    return System.currentTimeMillis() - lastShowTime >=interval;
}
```

```
/**
 * 启动页的时间间隔
 *
 * @return
 */
private static long getStartpageInterval() {
    int interval = CommonPreference.getInstance().getCmsShowStartPageInterval();
    if (interval < 0) {
        interval = CloudConfigExtra.getIntValue(INT_FUNC_TYPE, STRING_INSERT_SECTION,
                STRING_INSERT_INTERVAL_KEY, 45);
    }
    return interval * 60 * 1000;
}
```

#### 21，判断广告是否准备好。

```
if (!isAdReady) {
    InterstitialAdLoader.getInstance().reportPageTypeData(4);
    LauncherLog.releaseLog(TAG, " isAdReady  is false !!!");
    return null;
}
```

#### 22，展示Dialog。

```
private void showInsertDialog(InsertDataBean bean, int source) {
    if (bean == null) {
        return;
    }
    final int type = bean.type;
    if (!KOtherMessageHelper.isValidOtherMsgType(type)
            || LauncherSettingManager.getInstance(LauncherApplication.getLauncherApplication()).isSetDefaultDialogShowing()
            || ChargeActivity.isShowingChargeActivity()
            || BoostActivity.isIsShowingBoostActivity()
            || BatteryActivity.isIsShowingBatteryActivity()) {
        return;
    }

    if ((type == KMessageUtils.MESSAGE_WEATHER_TODAY || type == KMessageUtils.MESSAGE_WEATHER_FORECAST)) {
        if (WeatherDataCache.getInstance().isCacheExpire()) {
            InterstitialAdLoader.getInstance().reportPageTypeData(7);
            return;
        }
    }

    //如果是每日壁纸类型的话 如果预览图没有下载完成则不展示
    if (type == KMessageUtils.MESSAGE_DAILY_WALLPAPER) {
        if (!KOtherMessageHelper.isWallpaperThumbFileExist(LauncherApplication.getAppContext(), bean)) {
            LauncherLog.releaseLog(TAG, " wallpaper insert dialog : wallpaper file not exist.");
            InterstitialAdLoader.getInstance().reportPageTypeData(8);
            return;
        } else {
            KOtherMessageHelper.removeFailedList(bean);
        }
    }
    if (KMessageUtils.MESSAGE_CUSTOM_THEME == type) {
        if (!KOtherMessageHelper.isThemeThumbFileExist(LauncherApplication.getAppContext(), bean)) {
            LauncherLog.releaseLog(TAG, " custom theme  insert dialog : custom theme file not exist.");
            InterstitialAdLoader.getInstance().reportPageTypeData(9);
            return;
        } else {
            KOtherMessageHelper.removeFailedList(bean);
        }
    }

    if (isInsertDialogShowing()) {
        insertDialog.dismiss();
        insertDialog.removeCallbackForDailyWallpaper();
        insertDialog = null;
    }
    insertDialog = new InsertDialog(this, bean, source);
    String greetingMsg = "";
    if (KOtherMessageHelper.isGreetingMsgType(type)) {
        greetingMsg = KOtherMessageHelper.getGreetingByOrder();
    }
    insertDialog.setGreetingMsg(greetingMsg);
    showImmediatelyDialog(com.ksmobile.launcher.Dialog.Message.TYPE_INSERTDIALOG, new Callable<Boolean>() {
        @Override
        public Boolean call() {
            insertDialog.showByType(com.ksmobile.launcher.Dialog.Message.TYPE_INSERTDIALOG);
            return true;
        }
    });
    insertDialog.setOnDismissListener(this);
}

根据InsertDataBean查看出type数据那个弹窗
```



























