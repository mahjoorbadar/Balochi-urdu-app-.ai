# Balochi-urdu-app-.aihtml//
import 'package:flutter/material.dart';
import 'package:fl_chart/fl_chart.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'dart:convert';
import 'package:intl/intl.dart';
import 'package:timezone/data/latest.dart' as tz;
import 'package:timezone/timezone.dart' as tz;
import 'package:printing/printing.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  tz.initializeTimeZones();
  runApp(SugarControlApp());
}

class SugarControlApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'شوگر کنٹرول',
      theme: ThemeData(primarySwatch: Colors.teal, fontFamily: 'NotoNastaliq'),
      home: SugarHomePage(),
      debugShowCheckedModeBanner: false,
    );
  }
}

class SugarEntry {
  final double level;
  final DateTime date;

  SugarEntry(this.level, this.date);

  Map<String, dynamic> toJson() => {
        'level': level,
        'date': date.toIso8601String(),
      };

  factory SugarEntry.fromJson(Map<String, dynamic> json) =>
      SugarEntry(json['level'], DateTime.parse(json['date']));
}

enum FilterOption { all, today, week, month }

class SugarHomePage extends StatefulWidget {
  @override
  _SugarHomePageState createState() => _SugarHomePageState();
}

class _SugarHomePageState extends State<SugarHomePage> {
  List<SugarEntry> sugarEntries = [];
  final TextEditingController _controller = TextEditingController();
  late FlutterLocalNotificationsPlugin flutterLocalNotificationsPlugin;
  FilterOption selectedFilter = FilterOption.all;

  @override
  void initState() {
    super.initState();
    _initNotifications();
    _loadSugarLevels();
  }

  void _initNotifications() async {
    flutterLocalNotificationsPlugin = FlutterLocalNotificationsPlugin();

    const AndroidInitializationSettings initializationSettingsAndroid =
        AndroidInitializationSettings('@mipmap/ic_launcher');

    const InitializationSettings initializationSettings = InitializationSettings(
      android: initializationSettingsAndroid,
    );

    await flutterLocalNotificationsPlugin.initialize(initializationSettings);

    _scheduleDailyNotification(8, 0, 'دوائی لینا مت بھولیں');
    _scheduleDailyNotification(18, 0, 'براہ کرم اپنی شوگر چیک کریں');
  }

  void _scheduleDailyNotification(int hour, int minute, String message) async {
    await flutterLocalNotificationsPlugin.zonedSchedule(
      hour * 100 + minute,
      'یاد دہانی',
      message,
      tz.TZDateTime.local(DateTime.now().year, DateTime.now().month, DateTime.now().day, hour, minute).add(Duration(days: 1)),
      const NotificationDetails(
        android: AndroidNotificationDetails('daily_reminder', 'Daily Reminder', channelDescription: 'روزانہ کی یاد دہانی'),
      ),
      androidAllowWhileIdle: true,
      uiLocalNotificationDateInterpretation: UILocalNotificationDateInterpretation.absoluteTime,
      matchDateTimeComponents: DateTimeComponents.time,
    );
  }

  void _addSugarLevel() {
    final value = double.tryParse(_controller.text);
    if (value != null) {
      setState(() {
        sugarEntries.add(SugarEntry(value, DateTime.now()));
        _controller.clear();
        _saveSugarLevels();
      });
    }
  }

  void _saveSugarLevels() async {
    final prefs = await SharedPreferences.getInstance();
    final encoded = jsonEncode(sugarEntries.map((e) => e.toJson()).toList());
    prefs.setString('sugar_entries', encoded);
  }

  void _loadSugarLevels() async {
    final prefs = await SharedPreferences.getInstance();
    final data = prefs.getString('sugar_entries');
    if (data != null) {
      setState(() {
        sugarEntries = (jsonDecode(data) as List)
            .map((e) => SugarEntry.fromJson(e))
            .toList();
      });
    }
  }

  List<SugarEntry> get filteredEntries {
    final now = DateTime.now();
    switch (selectedFilter) {
      case FilterOption.today:
        return sugarEntries.where((e) => e.date.day == now.day && e.date.month == now.month && e.date.year == now.year).toList();
      case FilterOption.week:
        return sugarEntries.where((e) => e.date.isAfter(now.subtract(Duration(days: 7)))).toList();
      case FilterOption.month:
        return sugarEntries.where((e) => e.date.isAfter(now.subtract(Duration(days: 30)))).toList();
      case FilterOption.all:
      default:
        return sugarEntries;
    }
  }

  Future<void> _exportReportAsPDF() async {
    final pdf = pw.Document();
    final formatter = DateFormat('dd MMM yyyy, hh:mm a');

    pdf.addPage(
      pw.Page(
        build: (pw.Context context) {
          return pw.Directionality(
            textDirection: pw.TextDirection.rtl,
            child: pw.Column(
              crossAxisAlignment: pw.CrossAxisAlignment.start,
              children: [
                pw.Text('شوگر لیول ر
