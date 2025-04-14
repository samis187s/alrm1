import React, { useState, useEffect, useCallback, useRef } from 'react';
import {
  View,
  Text,
  FlatList,
  TouchableOpacity,
  TextInput,
  Switch,
  StyleSheet,
  PermissionsAndroid,
  Platform,
  Vibration,
  Animated,
  SafeAreaView,
  StatusBar,
  Dimensions,
  TimePickerAndroid,
  Alert,
} from 'react-native';
import Modal from 'react-native-modal';
import Icon from 'react-native-vector-icons/FontAwesome5';
import { accelerometer } from 'react-native-sensors';
import BackgroundTimer from 'react-native-background-timer';
import Sound from 'react-native-sound';
import MMKVStorage from 'react-native-mmkv';
import { useFocusEffect } from '@react-navigation/native';
import DateTimePicker from '@react-native-community/datetimepicker';
import Slider from '@react-native-community/slider';
import { LinearGradient } from 'expo-linear-gradient';
import * as Notifications from 'expo-notifications';
import * as Haptics from 'expo-haptics';

// Initialize MMKV storage
const storage = new MMKVStorage.Loader().initialize();

// Sound setup
Sound.setCategory('Playback');

// Configure notifications
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

const SENSITIVITY_THRESHOLD = 1.8; // Increased sensitivity for better step detection
const VIBRATION_PATTERN = [0, 500, 100, 500];
const DAYS_OF_WEEK = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

const App = () => {
  // State variables
  const [alarms, setAlarms] = useState([]);
  const [isAlarmModalVisible, setAlarmModalVisible] = useState(false);
  const [isRinging, setIsRinging] = useState(false);
  const [currentAlarm, setCurrentAlarm] = useState(null);
  const [stepsRemaining, setStepsRemaining] = useState(0);
  const [totalSteps, setTotalSteps] = useState(0);
  const [stepCount, setStepCount] = useState(0);
  const [selectedTime, setSelectedTime] = useState(new Date());
  const [steps, setSteps] = useState('20');
  const [showTimePicker, setShowTimePicker] = useState(false);
  const [repeatDays, setRepeatDays] = useState([false, false, false, false, false, false, false]);
  const [theme, setTheme] = useState('dark');
  const [alarmSoundOption, setAlarmSoundOption] = useState('default');
  const [snoozeEnabled, setSnoozeEnabled] = useState(true);
  const [editingAlarmIndex, setEditingAlarmIndex] = useState(null);
  
  // Animation refs
  const shakeAnimation = useRef(new Animated.Value(0)).current;
  const progressAnimation = useRef(new Animated.Value(0)).current;
  const bellAnimation = useRef(new Animated.Value(1)).current;
  
  // Sensor refs
  let lastAcceleration = useRef(null);
  const alarmSoundRef = useRef(null);
  const subscription = useRef(null);
  const notificationListener = useRef();
  const responseListener = useRef();
  
  // Load alarm sound
  useEffect(() => {
    alarmSoundRef.current = new Sound(
      alarmSoundOption === 'default' ? 'alarm.mp3' : `${alarmSoundOption}.mp3`,
      Sound.MAIN_BUNDLE,
      (error) => {
        if (error) console.log('Failed to load sound', error);
      }
    );
    
    return () => {
      if (alarmSoundRef.current) {
        alarmSoundRef.current.release();
      }
    };
  }, [alarmSoundOption]);

  // Initial setup
  useEffect(() => {
    loadAlarms();
    requestPermissions();
    setupNotifications();
    
    // Clean up on unmount
    return () => {
      BackgroundTimer.stopBackgroundTimer();
      unsubscribeFromSensors();
      if (notificationListener.current) {
        Notifications.removeNotificationSubscription(notificationListener.current);
      }
      if (responseListener.current) {
        Notifications.removeNotificationSubscription(responseListener.current);
      }
    };
  }, []);
  
  // Start background checking when app regains focus
  useFocusEffect(
    useCallback(() => {
      startBackgroundTask();
      return () => {
        BackgroundTimer.stopBackgroundTimer();
      };
    }, [])
  );
  
  // Update progress animation when steps change
  useEffect(() => {
    if (totalSteps > 0) {
      Animated.timing(progressAnimation, {
        toValue: (totalSteps - stepsRemaining) / totalSteps,
        duration: 300,
        useNativeDriver: false,
      }).start();
    }
  }, [stepsRemaining, totalSteps]);
  
  // Animate bell when alarm is ringing
  useEffect(() => {
    if (isRinging) {
      startBellAnimation();
    } else {
      bellAnimation.setValue(1);
    }
  }, [isRinging]);
  
  // Request necessary permissions
  const requestPermissions = async () => {
    try {
      if (Platform.OS === 'android') {
        await PermissionsAndroid.request(PermissionsAndroid.PERMISSIONS.ACTIVITY_RECOGNITION);
        await PermissionsAndroid.request(PermissionsAndroid.PERMISSIONS.VIBRATE);
      }
    } catch (err) {
      console.warn('Failed to request permissions:', err);
    }
  };
  
  // Set up notification handlers
  const setupNotifications = () => {
    notificationListener.current = Notifications.addNotificationReceivedListener((notification) => {
      // Handle incoming notification
    });
    
    responseListener.current = Notifications.addNotificationResponseReceivedListener((response) => {
      // Handle notification response (when user taps)
    });
  };
  
  // Bell animation (ring effect)
  const startBellAnimation = () => {
    Animated.loop(
      Animated.sequence([
        Animated.timing(bellAnimation, {
          toValue: 1.2,
          duration: 300,
          useNativeDriver: true,
        }),
        Animated.timing(bellAnimation, {
          toValue: 1,
          duration: 300,
          useNativeDriver: true,
        }),
      ])
    ).start();
  };
  
  // Shake animation for remaining steps
  const startShakeAnimation = () => {
    Animated.sequence([
      Animated.timing(shakeAnimation, { toValue: 10, duration: 100, useNativeDriver: true }),
      Animated.timing(shakeAnimation, { toValue: -10, duration: 100, useNativeDriver: true }),
      Animated.timing(shakeAnimation, { toValue: 10, duration: 100, useNativeDriver: true }),
      Animated.timing(shakeAnimation, { toValue: 0, duration: 100, useNativeDriver: true })
    ]).start();
  };
  
  // Load alarms from storage
  const loadAlarms = () => {
    try {
      const savedAlarms = storage.getString('alarms');
      setAlarms(savedAlarms ? JSON.parse(savedAlarms) : []);
    } catch (error) {
      console.error('Failed to load alarms:', error);
      setAlarms([]);
    }
  };
  
  // Save alarms to storage
  const saveAlarms = (newAlarms) => {
    try {
      storage.setString('alarms', JSON.stringify(newAlarms));
      setAlarms(newAlarms);
      scheduleNotifications(newAlarms);
    } catch (error) {
      console.error('Failed to save alarms:', error);
      Alert.alert('Error', 'Failed to save your alarms. Please try again.');
    }
  };
  
  // Schedule notifications for each enabled alarm
  const scheduleNotifications = async (alarmList) => {
    await Notifications.cancelAllScheduledNotificationsAsync();
    
    alarmList.forEach(async (alarm) => {
      if (!alarm.enabled) return;
      
      try {
        const [hours, minutes] = alarm.time.split(':').map(Number);
        
        // For each selected day or just once
        const daysToSchedule = alarm.repeatDays.some(day => day) 
          ? alarm.repeatDays.map((selected, index) => selected ? index : -1).filter(day => day !== -1)
          : [new Date().getDay()];
          
        for (const day of daysToSchedule) {
          const trigger = new Date();
          trigger.setDate(trigger.getDate() + (day + 7 - trigger.getDay()) % 7);
          trigger.setHours(hours);
          trigger.setMinutes(minutes);
          trigger.setSeconds(0);
          
          // If today and time already passed, schedule for next week
          if (trigger < new Date()) {
            trigger.setDate(trigger.getDate() + 7);
          }
          
          await Notifications.scheduleNotificationAsync({
            content: {
              title: 'StepUp Alarm',
              body: `Time to get up! ${alarm.steps} steps to disable.`,
              sound: true,
              priority: Notifications.AndroidNotificationPriority.HIGH,
              data: { alarmId: alarm.id },
            },
            trigger,
          });
        }
      } catch (err) {
        console.error('Error scheduling notification:', err);
      }
    });
  };
  
  // Add a new alarm
  const addAlarm = () => {
    const hours = selectedTime.getHours().toString().padStart(2, '0');
    const minutes = selectedTime.getMinutes().toString().padStart(2, '0');
    const timeString = `${hours}:${minutes}`;
    const formattedTime = selectedTime.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    
    if (editingAlarmIndex !== null) {
      // Edit existing alarm
      const newAlarms = [...alarms];
      newAlarms[editingAlarmIndex] = {
        ...newAlarms[editingAlarmIndex],
        time: timeString,
        displayTime: formattedTime,
        steps: parseInt(steps),
        repeatDays: [...repeatDays],
        snoozeEnabled,
        alarmSound: alarmSoundOption,
      };
      saveAlarms(newAlarms);
      setEditingAlarmIndex(null);
    } else {
      // Add new alarm
      const newAlarm = {
        id: Date.now().toString(),
        time: timeString,
        displayTime: formattedTime,
        steps: parseInt(steps),
        enabled: true,
        repeatDays: [...repeatDays],
        createdAt: Date.now(),
        snoozeEnabled,
        alarmSound: alarmSoundOption,
      };
      saveAlarms([...alarms, newAlarm]);
    }
    
    resetModalState();
  };
  
  // Reset modal state after adding/editing
  const resetModalState = () => {
    setAlarmModalVisible(false);
    setSelectedTime(new Date());
    setSteps('20');
    setRepeatDays([false, false, false, false, false, false, false]);
    setSnoozeEnabled(true);
    setAlarmSoundOption('default');
    setEditingAlarmIndex(null);
  };
  
  // Toggle alarm enabled/disabled
  const toggleAlarm = (index) => {
    const newAlarms = [...alarms];
    newAlarms[index].enabled = !newAlarms[index].enabled;
    saveAlarms(newAlarms);
    
    // Provide haptic feedback
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
  };
  
  // Delete an alarm
  const deleteAlarm = (index) => {
    Alert.alert(
      "Delete Alarm",
      "Are you sure you want to delete this alarm?",
      [
        {
          text: "Cancel",
          style: "cancel"
        },
        { 
          text: "Delete", 
          onPress: () => {
            const newAlarms = alarms.filter((_, i) => i !== index);
            saveAlarms(newAlarms);
            Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
          },
          style: "destructive"
        }
      ]
    );
  };
  
  // Edit an existing alarm
  const editAlarm = (index) => {
    const alarm = alarms[index];
    const [hours, minutes] = alarm.time.split(':').map(Number);
    const timeDate = new Date();
    timeDate.setHours(hours);
    timeDate.setMinutes(minutes);
    
    setSelectedTime(timeDate);
    setSteps(alarm.steps.toString());
    setRepeatDays(alarm.repeatDays || [false, false, false, false, false, false, false]);
    setSnoozeEnabled(alarm.snoozeEnabled !== undefined ? alarm.snoozeEnabled : true);
    setAlarmSoundOption(alarm.alarmSound || 'default');
    setEditingAlarmIndex(index);
    setAlarmModalVisible(true);
  };
  
  // Subscribe to accelerometer for step counting
  const startStepCounting = () => {
    unsubscribeFromSensors(); // Clean up any existing subscription
    
    subscription.current = accelerometer.subscribe(({ x, y, z }) => {
      if (!isRinging) return;
      
      const magnitude = Math.sqrt(x * x + y * y + z * z);
      
      if (lastAcceleration.current !== null) {
        const delta = Math.abs(magnitude - lastAcceleration.current);
        
        if (delta > SENSITIVITY_THRESHOLD) {
          setStepCount((prev) => {
            const newCount = prev + 1;
            const newRemaining = Math.max(0, totalSteps - newCount);
            
            if (newRemaining !== stepsRemaining) {
              // Vibrate on each step
              Vibration.vibrate(100);
              Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
              
              // Shake animation for step count
              startShakeAnimation();
            }
            
            setStepsRemaining(newRemaining);
            
            if (newRemaining === 0) {
              stopAlarm();
              Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
            }
            
            return newCount;
          });
        }
      }
      
      lastAcceleration.current = magnitude;
    });
  };
  
  // Unsubscribe from accelerometer
  const unsubscribeFromSensors = () => {
    if (subscription.current) {
      subscription.current.unsubscribe();
      subscription.current = null;
    }
  };
  
  // Stop the currently ringing alarm
  const stopAlarm = () => {
    if (alarmSoundRef.current) {
      alarmSoundRef.current.stop();
    }
    
    Vibration.cancel();
    setIsRinging(false);
    setStepsRemaining(0);
    setStepCount(0);
    lastAcceleration.current = null;
    unsubscribeFromSensors();
    
    if (currentAlarm !== null) {
      const newAlarms = [...alarms];
      // Only disable one-time alarms, keep repeating ones
      if (!newAlarms[currentAlarm].repeatDays.some(day => day)) {
        newAlarms[currentAlarm].enabled = false;
        saveAlarms(newAlarms);
      }
    }
    
    setCurrentAlarm(null);
  };
  
  // Snooze the current alarm
  const snoozeAlarm = () => {
    stopAlarm();
    
    // Schedule a notification for 5 minutes later
    const snoozeTime = new Date();
    snoozeTime.setMinutes(snoozeTime.getMinutes() + 5);
    
    Notifications.scheduleNotificationAsync({
      content: {
        title: 'StepUp Alarm - Snoozed',
        body: 'Your alarm was snoozed for 5 minutes.',
        sound: true,
        data: { snooze: true },
      },
      trigger: { date: snoozeTime },
    });
    
    Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
    Alert.alert('Alarm Snoozed', 'Alarm will ring again in 5 minutes');
  };
  
  // Check for alarms that should ring
  const checkAlarms = () => {
    if (isRinging) return;
    
    const now = new Date();
    const nowHours = now.getHours().toString().padStart(2, '0');
    const nowMinutes = now.getMinutes().toString().padStart(2, '0');
    const nowTime = `${nowHours}:${nowMinutes}`;
    const today = now.getDay();
    
    alarms.forEach((alarm, index) => {
      if (!alarm.enabled) return;
      
      const shouldRingToday = !alarm.repeatDays || 
        !alarm.repeatDays.some(day => day) || 
        alarm.repeatDays[today];
      
      if (shouldRingToday && alarm.time === nowTime && !isRinging) {
        setIsRinging(true);
        setCurrentAlarm(index);
        setTotalSteps(alarm.steps);
        setStepsRemaining(alarm.steps);
        
        // Play sound
        if (alarmSoundRef.current) {
          alarmSoundRef.current.setNumberOfLoops(-1); // Loop indefinitely
          alarmSoundRef.current.play((success) => {
            if (!success) {
              console.log('Sound playback failed');
            }
          });
        }
        
        // Start vibration
        Vibration.vibrate(VIBRATION_PATTERN, true);
        
        // Start step counting
        startStepCounting();
      }
    });
  };
  
  // Start background checking for alarms
  const startBackgroundTask = () => {
    BackgroundTimer.stopBackgroundTimer(); // Stop any existing timer
    
    BackgroundTimer.runBackgroundTimer(() => {
      checkAlarms();
    }, 1000);
  };
  
  // Handle time picker changes
  const onTimeChange = (event, selectedDate) => {
    setShowTimePicker(Platform.OS === 'ios');
    if (selectedDate) {
      setSelectedTime(selectedDate);
    }
  };
  
  // Toggle repeat day selection
  const toggleDay = (index) => {
    const newRepeatDays = [...repeatDays];
    newRepeatDays[index] = !newRepeatDays[index];
    setRepeatDays(newRepeatDays);
  };
  
  // Individual alarm item component
  const AlarmItem = ({ alarm, index }) => {
    const isRepeating = alarm.repeatDays && alarm.repeatDays.some(day => day);
    
    return (
      <TouchableOpacity 
        style={[
          styles.alarmItem, 
          !alarm.enabled && styles.alarmItemDisabled
        ]}
        onPress={() => editAlarm(index)}
        activeOpacity={0.7}
      >
        <View style={styles.alarmItemContent}>
          <View style={styles.alarmTimeContainer}>
            <Text style={[styles.alarmTime, !alarm.enabled && styles.alarmTimeDisabled]}>
              {alarm.displayTime}
            </Text>
            
            {isRepeating && (
              <View style={styles.daysContainer}>
                {DAYS_OF_WEEK.map((day, idx) => (
                  <Text 
                    key={day} 
                    style={[
                      styles.dayIndicator,
                      alarm.repeatDays[idx] && styles.dayIndicatorActive,
                      !alarm.enabled && styles.dayIndicatorDisabled
                    ]}
                  >
                    {day}
                  </Text>
                ))}
              </View>
            )}
          </View>
          
          <View style={styles.alarmDetails}>
            <Text style={styles.alarmSteps}>
              <Icon name="walking" size={14} /> {alarm.steps} steps to disable
            </Text>
            
            {alarm.snoozeEnabled && (
              <Text style={styles.snoozeIndicator}>
                <Icon name="bell-slash" size={12} /> Snooze enabled
              </Text>
            )}
          </View>
        </View>
        
        <View style={styles.alarmControls}>
          <Switch
            value={alarm.enabled}
            onValueChange={() => toggleAlarm(index)}
            trackColor={{ false: '#374151', true: '#4f46e5' }}
            thumbColor={alarm.enabled ? '#6366f1' : '#9ca3af'}
          />
          
          <TouchableOpacity 
            onPress={() => deleteAlarm(index)} 
            style={styles.deleteBtn}
            hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
          >
            <Icon name="trash-alt" size={18} color="#ef4444" />
          </TouchableOpacity>
        </View>
      </TouchableOpacity>
    );
  };

  // Empty list component
  const EmptyListComponent = () => (
    <View style={styles.emptyContainer}>
      <Icon name="bell-slash" size={60} color="#4b5563" />
      <Text style={styles.noAlarms}>No alarms set</Text>
      <Text style={styles.noAlarmsSubtext}>
        Tap the "Add New Alarm" button to create your first wake-up challenge.
      </Text>
    </View>
  );

  return (
    <SafeAreaView style={styles.safeArea}>
      <StatusBar barStyle="light-content" backgroundColor="#111827" />
      <LinearGradient
        colors={['#111827', '#1e293b']}
        style={styles.container}
      >
        <View style={styles.appHeader}>
          <Text style={styles.appTitle}>StepUp Alarm</Text>
          <Text style={styles.appSubtitle}>Walk to stop. Sleep no more.</Text>
        </View>
        
        <TouchableOpacity 
          style={styles.setAlarmBtn} 
          onPress={() => setAlarmModalVisible(true)}
          activeOpacity={0.8}
        >
          <Icon name="plus" size={18} color="white" />
          <Text style={styles.setAlarmText}> Add New Alarm</Text>
        </TouchableOpacity>
        
        <FlatList
          data={alarms}
          renderItem={({ item, index }) => <AlarmItem alarm={item} index={index} />}
          keyExtractor={(item) => item.id || item.createdAt.toString()}
          style={styles.alarmList}
          contentContainerStyle={alarms.length === 0 ? styles.emptyListContainer : null}
          ListEmptyComponent={<EmptyListComponent />}
          showsVerticalScrollIndicator={false}
        />

        {/* Create/Edit Alarm Modal */}
        <Modal 
          isVisible={isAlarmModalVisible} 
          onBackdropPress={() => setAlarmModalVisible(false)}
          backdropTransitionOutTiming={0}
          animationIn="fadeIn"
          animationOut="fadeOut"
          backdropOpacity={0.7}
          style={styles.modal}
        >
          <View style={styles.modalContent}>
            <Text style={styles.modalTitle}>
              {editingAlarmIndex !== null ? 'Edit Alarm' : 'Create New Alarm'}
            </Text>
            
            <View style={styles.timeSelectorContainer}>
              <TouchableOpacity
                style={styles.timeSelector}
                onPress={() => setShowTimePicker(true)}
              >
                <Icon name="clock" size={24} color="#6366f1" style={styles.inputIcon} />
                <Text style={styles.timeDisplay}>
                  {selectedTime.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
                </Text>
              </TouchableOpacity>
            </View>
            
            {showTimePicker && (
              <DateTimePicker
                value={selectedTime}
                mode="time"
                is24Hour={false}
                display="default"
                onChange={onTimeChange}
              />
            )}
            
            <View style={styles.stepsSelectorContainer}>
              <Text style={styles.inputLabel}>
                <Icon name="walking" size={16} color="#6366f1" /> Steps to Disable: {steps}
              </Text>
              <Slider
                style={styles.slider}
                minimumValue={5}
                maximumValue={100}
                step={5}
                value={parseInt(steps)}
                onValueChange={(value) => setSteps(value.toString())}
                minimumTrackTintColor="#6366f1"
                maximumTrackTintColor="#374151"
                thumbTintColor="#818cf8"
              />
            </View>
            
            <View style={styles.repeatContainer}>
              <Text style={styles.inputLabel}>Repeat</Text>
              <View style={styles.daysSelector}>
                {DAYS_OF_WEEK.map((day, index) => (
                  <TouchableOpacity
                    key={day}
                    style={[
                      styles.dayButton,
                      repeatDays[index] && styles.dayButtonSelected,
                    ]}
                    onPress={() => toggleDay(index)}
                  >
                    <Text style={repeatDays[index] ? styles.dayTextSelected : styles.dayText}>
                      {day}
                    </Text>
                  </TouchableOpacity>
                ))}
              </View>
            </View>
            
            <View style={styles.optionRow}>
              <Text style={styles.inputLabel}>Snooze</Text>
              <Switch
                value={snoozeEnabled}
                onValueChange={setSnoozeEnabled}
                trackColor={{ false: '#374151', true: '#4f46e5' }}
                thumbColor={snoozeEnabled ? '#6366f1' : '#9ca3af'}
              />
            </View>
            
            <View style={styles.optionRow}>
              <Text style={styles.inputLabel}>Alarm Sound</Text>
              <View style={styles.soundSelector}>
                <TouchableOpacity
                  style={[
                    styles.soundOption,
                    alarmSoundOption === 'default' && styles.soundOptionSelected,
                  ]}
                  onPress={() => setAlarmSoundOption('default')}
                >
                  <Text style={styles.soundOptionText}>Default</Text>
                </TouchableOpacity>
                
                <TouchableOpacity
                  style={[
                    styles.soundOption,
                    alarmSoundOption === 'gentle' && styles.soundOptionSelected,
                  ]}
                  onPress={() => setAlarmSoundOption('gentle')}
                >
                  <Text style={styles.soundOptionText}>Gentle</Text>
                </TouchableOpacity>
                
                <TouchableOpacity
                  style={[
                    styles.soundOption,
                    alarmSoundOption === 'urgent' && styles.soundOptionSelected,
                  ]}
                  onPress={() => setAlarmSoundOption('urgent')}
                >
                  <Text style={styles.soundOptionText}>Urgent</Text>
                </TouchableOpacity>
              </View>
            </View>
            
            <View style={styles.modalActions}>
              <TouchableOpacity 
                onPress={() => setAlarmModalVisible(false)} 
                style={styles.btnSecondary}
              >
                <Text style={styles.btnText}>Cancel</Text>
              </TouchableOpacity>
              
              <TouchableOpacity 
                onPress={addAlarm} 
                style={styles.btnPrimary}
              >
                <Text style={styles.btnText}>Save</Text>
              </TouchableOpacity>
            </View>
          </View>
        </Modal>

        {/* Ringing Alarm Modal */}
        {currentAlarm !== null && (
          <Modal 
            isVisible={isRinging}
            backdropOpacity={0.9}
            animationIn="bounceIn"
            style={styles.alarmModal}
          >
            <View style={styles.ringingModalContent}>
              <Animated.View
                style={{
                  transform: [
                    { scale: bellAnimation }
                  ]
                }}
              >
                <Icon name="bell" size={80} color="#6366f1" />
              </Animated.View>
              
              <Text style={styles.ringingTime}>
                {alarms[currentAlarm]?.displayTime}
              </Text>
              
              <View style={styles.stepsContainer}>
                <Text style={styles.stepsTitle}>Steps remaining to disable:</Text>
                
                <Animated.Text 
                  style={[
                    styles.stepsRemaining,
                    {
                      transform: [{ translateX: shakeAnimation }]
                    }
                  ]}
                >
                  {stepsRemaining}
                </Animated.Text>
                
                <View style={styles.progressBarContainer}>
                  <View style={styles.progressBar}>
                    <Animated.View
                      style={[
                        styles.progress,
                        {
                          width: progressAnimation.interpolate({
                            inputRange: [0, 1],
                            outputRange: ['0%', '100%']
                          })
                        },
                      ]}
                    />
                  </View>
                </View>
                
                <Text style={styles.stepsHint}>
                  <Icon name="walking" size={16} /> Start walking to stop the alarm
                </Text>
                
                {alarms[currentAlarm]?.snoozeEnabled && (
                  <TouchableOpacity 
                    style={styles.snoozeButton}
                    onPress={snoozeAlarm}
                  >
                    <Icon name="stopwatch" size={16} color="#fff" />
                    <Text style={styles.snoozeButtonText}>Snooze (5 min)</Text>
                  </TouchableOpacity>
                )}
              </View>
            </View>
          </Modal>
        )}
      </LinearGradient>
    </SafeAreaView>
  );
};

const { width } = Dimensions.get('window');

const styles = StyleSheet.create({
  safeArea: {
    flex: 1,
    backgroundColor: '#111827',
  },
  container: {
    flex: 1,
    padding: 16,
  },
  appHeader: {
    alignItems: 'center',
    marginTop: 20,
    marginBottom: 32,
    borderBottomWidth: 1,
    borderBottomColor: 'rgba(255, 255, 255, 0.1)',
    paddingBottom: 24,
  },
  appTitle: {
    fontSize: 36,
    fontWeight: '800 
