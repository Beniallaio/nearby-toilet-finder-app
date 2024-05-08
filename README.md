# nearby-toilet-finder-app
Main.dart code
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:smart_toire/screens/finder_view.dart';
import 'package:smart_toire/screens/landing_view.dart';
import 'package:smart_toire/screens/splash_view.dart';
import 'package:smart_toire/utils/app_strings.dart';
import 'package:smart_toire/utils/app_style.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  SystemChrome.setPreferredOrientations([DeviceOrientation.portraitUp])
      .then((value) => runApp(MyApp()));
}

class MyApp extends StatelessWidget {

  final routes = <String, WidgetBuilder>{
    SplashView.tag: (context) => const SplashView(),
    LandingView.tag: (context) => const LandingView(),
    FinderView.tag: (context) => const FinderView(),
  };

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: AppStrings.Lbl_AppName,
      theme: ThemeData(
        primarySwatch: MyStyle.primaryColor,
        fontFamily: "OpenSans",
      ),
      home: SplashView(),
      routes: routes,
//      onGenerateRoute: (settings){
//        final Division division = settings.arguments;
//        if (settings.name == DivisionPage.tag) {
//          return MaterialPageRoute(
//            builder: (context) {
//              return division;
//            }
//          );
//        },
//      },
    );
  }
}
Findar view.dart
import 'dart:async';

import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:smart_toire/data_model/device_model.dart';
import 'package:smart_toire/db_helper/db_device_helper.dart';
import 'package:smart_toire/rtb_libs/ui/choice.dart';
import 'package:smart_toire/utils/app_assets.dart';
import 'package:smart_toire/utils/app_logger.dart';
import 'package:smart_toire/utils/app_strings.dart';
import 'package:smart_toire/utils/app_style.dart';
import 'package:smart_toire/utils/app_toast.dart';
import 'package:smart_toire/utils/status_dialog.dart';
import 'package:flutter/material.dart';

class FinderView extends StatefulWidget {
  static String tag = 'finder-screen';
  const FinderView({super.key});
  @override
  _FinderViewState createState() => new _FinderViewState();
}

class _FinderViewState extends State<FinderView> with WidgetsBindingObserver {

  List<Choice> appbarContextMenu = [];
  late Map<String, Marker> _markers = {};
  List<DeviceModel> deviceList = <DeviceModel>[];
  DeviceModel? selectedDevice;

  Widget getStar(int index, DeviceModel? model) {
    var star = AppAssets.IconStar3;
    if (index < (model?.rating ?? 0) ) {
      star = AppAssets.IconStar;
    } else if ((model?.rating?? 0) - index > -1) {
      star = AppAssets.IconStar2;
    }
    return SizedBox(
      width: 16,
      height: 16,
      child: Image.asset(
        star,
        fit: BoxFit.fitWidth,
      ),
    );
  }

  @override
  void initState () {
    //Show status dialog
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    WidgetsBinding.instance
        .addPostFrameCallback((_) => afterFirstFrameRender(context));
    setState(() {
      appbarContextMenu.add(Choice(title: AppStrings.Menu_Sign_out, icon: Icons.exit_to_app));
    });
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    setState(() {
      if(state == AppLifecycleState.resumed) {
        //TODO: Auth based handling
      }
    });
  }

  // Callback After the first frame render completed
  Future<void> afterFirstFrameRender (BuildContext context) async {
    var devices = await DBDeviceHelper(null).getAllDevice();
    for(DeviceModel dev in devices ?? []) {
      Logger.Log("=====> ${dev.dev_id}");
    }

    updatePinWithFilter(context,"","");
    // await StatusDialog.displayProgressDialog(context,AppStrings.Loader_Loading);
  }

  LatLngBounds _boundsFromLatLngList(List<LatLng> list) {
    assert(list.isNotEmpty);
    double? x0, x1, y0, y1;
    for (LatLng latLng in list) {
      if (x0 == null) {
        x0 = x1 = latLng.latitude;
        y0 = y1 = latLng.longitude;
      } else {
        if (latLng.latitude > (x1 ?? 0)) x1 = latLng.latitude;
        if (latLng.latitude < x0) x0 = latLng.latitude;
        if (latLng.longitude > (y1 ?? 0)) y1 = latLng.longitude;
        if (latLng.longitude < (y0 ?? double.infinity)) y0 = latLng.longitude;
      }
    }
    return LatLngBounds(
      northeast: LatLng(x1 ?? 0, y1 ?? 0),
      southwest: LatLng(x0 ?? 0, y0 ?? 0),
    );
  }

  Future<void> updatePinWithFilter(
      BuildContext context,
      String zoneArg,
      String wardArg,
      ) async {
    await StatusDialog.displayProgressDialog(context, "Loading devices");
    List<DeviceModel>? devices = await DBDeviceHelper(null)
        .getAllDevice();
    // if (response.isRight) {
    //   var devices = response.right;
      Map<String, Marker> markers = {};
      List<LatLng> positions = [];
      for (DeviceModel office in devices ?? []) {
        if (office.loc != null &&
            office.loc?.latitude != 0.0 &&
            office.loc?.longitude != 0.0) {
          var mapPin = await getMapPin(office);
          final marker = Marker(
              markerId:
              MarkerId(office.dev_id ?? "dev-${devices?.indexOf(office)}"),
              position: LatLng(
                  office.loc?.latitude ?? 0.0, office.loc?.longitude ?? 0.0),
              infoWindow: InfoWindow(
                title: office.name,
                snippet: office.addr,
              ),
              icon: mapPin,
              onTap: () async {
                setState(() {
                  selectedDevice = office;
                });
                // print("On click device");
                // print("SLC ID ${office.uid}");
                // List<LatLng> lampPositions = [];

              });
          markers[office.dev_id ?? "dev-${devices?.indexOf(office)}"] = marker;
          positions.add(LatLng(
              office.loc?.latitude ?? 0.0, office.loc?.longitude ?? 0.0));
        }
      }
      setState(() {
        // area = areaData;
        deviceList = devices ?? [];
        _markers.clear();
        _markers = markers;
      });

      if (positions.isNotEmpty) {
        // Calculate the bounds to fit all the markers
        var bounds = _boundsFromLatLngList(positions);
        // Create the camera update with the bounds calculated
        CameraUpdate u2 = CameraUpdate.newLatLngBounds(bounds, 50);
        // Animate the camera to update
        GoogleMapController googleMapController = await _controller.future;
        await googleMapController.animateCamera(u2);
      }
      if (positions.isEmpty) {
        AppToast().showToast("No device found", "FAIL");
      } else {
        AppToast().showToast("${positions.length} device found", "SUCCESS");
      }
    // }
    // else {
    //   AppToast().showToast("Unable to process the request", "FAIL");
    // }
    StatusDialog.hideProgressDialog();
  }

  Future<BitmapDescriptor> getMapPin(DeviceModel device) async {

    return await BitmapDescriptor.fromAssetImage(
      const ImageConfiguration(size: Size(10, 10)),
      getDevicePin(device),
    );

  }

  String getDevicePin(DeviceModel device) {
    // switch (device.status) {
    //   case 1:
        if (device.status == false) {
          return AppAssets.IconLampGray;
        } else {
          return AppAssets.IconLampGreen;
        }
      // case 0:
      // case 2:
        return AppAssets.IconLampRed;
    // }
    return AppAssets.IconLampGray;
  }

  final Completer<GoogleMapController> _controller =
  Completer<GoogleMapController>();

  static const CameraPosition _kGooglePlex = CameraPosition(
    //target: LatLng(37.42796133580664, -122.085749655962),
    target: LatLng(0, 0),
    zoom: 14.4746,
  );


  @override
  Widget build(BuildContext context) {

    return Scaffold(
      backgroundColor: MyStyle.backgroundLight,
      // appBar: AppBar(
      //   title: Text(AppStrings.APP_NAME),
      //   actions: <Widget>[
      //     PopupMenuButton<Choice>(
      //       onSelected: onSelectAppbarContextMenu,
      //       itemBuilder: (BuildContext context) {
      //         return appbarContextMenu.map((Choice choice) {
      //           return PopupMenuItem<Choice>(
      //             value: choice,
      //               child: Row(
      //                 children: <Widget>[
      //                   Text(choice.title, style: TextStyle(fontSize: MyStyle.lblFontSizeContextMenu),),
      //                   Expanded(child: Icon(choice.icon, color: MyStyle.IcColorContextMenu,),)
      //                 ],
      //               ),
      //           );
      //         }).toList();
      //       },
      //     ),
      //   ],
      // ),
      body: Stack(
        children: [
          Column(children: [
            Expanded(
                child: GoogleMap(
                  mapType: MapType.hybrid,
                  initialCameraPosition: _kGooglePlex,
                  onMapCreated: (GoogleMapController controller) {
                    _controller.complete(controller);
                  },
                  markers: _markers.values.toSet(),
                )),
            Visibility(
              visible: selectedDevice != null,
              // child: Expanded(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.end,
                  crossAxisAlignment: CrossAxisAlignment.center,
                  children: [
                    Container(
                      width: MediaQuery.of(context).size.width,
                      decoration: BoxDecoration(
                          shape: BoxShape.rectangle,
                          color: MyStyle.textWhite,
                          borderRadius: const BorderRadius.only(topLeft: Radius.circular(MyStyle.borderRadius),topRight: Radius.circular(MyStyle.borderRadius)),
                          boxShadow: [
                            BoxShadow(
                              color: MyStyle.shadowDarkGray.withOpacity(0.5),
                              spreadRadius: 5,
                              blurRadius: 7,
                              offset: Offset(0, 3), // changes position of shadow
                            ),
                          ]
                      ),
                      child: Stack(
                        children: [
                          Column(
                              mainAxisAlignment: MainAxisAlignment.center,
                              crossAxisAlignment: CrossAxisAlignment.center,
                              children: [
                                const SizedBox(height: 20),
                                Text("${selectedDevice?.addr}",
                                    style: const TextStyle(
                                      fontSize: 18,
                                    ),
                                ),
                                const SizedBox(height: 10),
                                Row(children: [
                                  SizedBox(width: 20),
                                  Text("${selectedDevice?.rating}",
                                    style: const TextStyle(
                                      fontSize: 18,
                                      color: MyStyle.borderDarkGray
                                    ),
                                  ),
                                  SizedBox(width: 10),
                                  ///Rating
                                  getStar(1, selectedDevice),
                                  SizedBox(width: 3),
                                  getStar(2, selectedDevice),
                                  SizedBox(width: 3),
                                  getStar(3, selectedDevice),
                                  SizedBox(width: 3),
                                  getStar(4, selectedDevice),
                                  SizedBox(width: 3),
                                  getStar(5, selectedDevice),
                                  SizedBox(width: 10),
                                  Text("(${selectedDevice?.rating_count})",
                                    style: const TextStyle(
                                        fontSize: 18,
                                        color: MyStyle.borderDarkGray
                                    ),
                                  )
                                ]),
                                const SizedBox(height: 20),
                                Row(
                                  crossAxisAlignment: CrossAxisAlignment.center,
                                  mainAxisAlignment: MainAxisAlignment.center,
                                  children: [
                                    Expanded(
                                        child: Center(
                                            child: SizedBox(
                                              width: 150,
                                              child:ElevatedButton(
                                                onPressed: () {
                                                  // Navigator.of(context).pop();
                                                },
                                                style: ElevatedButton.styleFrom( // styling the button
                                                  shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(20)),
                                                  padding: EdgeInsets.only(top: 10, bottom: 10),
                                                  backgroundColor: Colors.white, // Button color
                                                  foregroundColor: Colors.cyan, // Splash color
                                                ),
                                                child: const Text("Rate Us",
                                                  style: TextStyle(
                                                      fontSize: 24,
                                                      fontWeight: FontWeight.bold,
                                                      color: Colors.purple
                                                  ),),
                                              ),
                                            ))),
                                    Expanded(
                                        child: Center(
                                            child: SizedBox(
                                              width: 150,
                                              child:ElevatedButton(
                                                onPressed: () {
                                                  // Navigator.of(context)
                                                  //     .pushNamed(FinderView.tag);
                                                },
                                                style: ElevatedButton.styleFrom( // styling the button
                                                  shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(20)),
                                                  padding: EdgeInsets.only(top: 10, bottom: 10),
                                                  backgroundColor: MyStyle.purple, // Button color
                                                  foregroundColor: Colors.cyan, // Splash color
                                                ),
                                                child: const Text("Share",
                                                  style: TextStyle(
                                                      fontSize: 24,
                                                      fontWeight: FontWeight.bold,
                                                      color: MyStyle.textWhite
                                                  ),),
                                              ),
                                            ))),
                                  ],
                                ),
                                const SizedBox(height: 30),
                              ]),
                          Row(
                            mainAxisAlignment: MainAxisAlignment.end,
                            crossAxisAlignment: CrossAxisAlignment.end,
                            children: [
                              ElevatedButton(
                                  onPressed: () {
                                    setState(() {
                                      selectedDevice = null;
                                    });
                                  },
                                  style: ElevatedButton.styleFrom( // styling the button
                                    shape: CircleBorder(),
                                    padding: EdgeInsets.all(5),
                                    backgroundColor: MyStyle.grayLight, // Button color
                                    foregroundColor: Colors.cyan, // Splash color
                                  ),
                                  child: const Icon(Icons.close, color: MyStyle.textDark)
                              )
                            ],
                          )
                        ],
                      ),
                    ),
                    // )
                  ],
                ),
              // ),
            ),
          ]),

        ],
      )
      // GoogleMap(
      //   mapType: MapType.normal,
      //   initialCameraPosition: _kGooglePlex,
      //   onMapCreated: (GoogleMapController controller) {
      //     _controller.complete(controller);
      //   },
      //   markers: _markers.values.toSet(),
      // )
    );
  }

  void onSelectAppbarContextMenu(Choice choice) {
    // Causes the app to rebuild with the new _selectedChoice.
    setState(() {
      switch (choice.title) {
        case AppStrings.Menu_Sign_out:
          {

            break;
          }
      }
    });
  }
}

Landing view.dart
import 'package:smart_toire/rtb_libs/ui/choice.dart';
import 'package:smart_toire/screens/finder_view.dart';
import 'package:smart_toire/utils/app_assets.dart';
import 'package:smart_toire/utils/app_strings.dart';
import 'package:smart_toire/utils/app_style.dart';
import 'package:smart_toire/utils/status_dialog.dart';
import 'package:flutter/material.dart';

class LandingView extends StatefulWidget {
  static String tag = 'landing-screen';
  const LandingView({super.key});
  @override
  _LandingViewState createState() => new _LandingViewState();
}

class _LandingViewState extends State<LandingView> with WidgetsBindingObserver {

  List<Choice> appbarContextMenu = [];

  @override
  void initState () {
    //Show status dialog
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    WidgetsBinding.instance
        .addPostFrameCallback((_) => afterFirstFrameRender(context));
    setState(() {
      appbarContextMenu.add(Choice(title: AppStrings.Menu_Sign_out, icon: Icons.exit_to_app));
    });
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    setState(() {
      if(state == AppLifecycleState.resumed) {
        //TODO: Auth based handling
      }
    });
  }

  // Callback After the first frame render completed
  Future<void> afterFirstFrameRender (BuildContext context) async {
    // await StatusDialog.displayProgressDialog(context,AppStrings.Loader_Loading);
  }

  @override
  Widget build(BuildContext context) {

    return Scaffold(
      backgroundColor: MyStyle.textWhite,
      body: Column(
        children: <Widget>[
          Expanded(
            child: Container(
              width: MediaQuery.of(context).size.width,
              child: Image.asset(
                AppAssets.ImageBanner2,
                fit: BoxFit.fitWidth,
              ),
            ),
          ),
          Container(
            width: double.infinity,
            decoration: BoxDecoration(
                shape: BoxShape.rectangle,
                color: MyStyle.purple,
                borderRadius: const BorderRadius.only(topLeft: Radius.circular(MyStyle.borderRadius),topRight: Radius.circular(MyStyle.borderRadius)),
                boxShadow: [
                  BoxShadow(
                    color: MyStyle.shadowDarkGray.withOpacity(0.5),
                    spreadRadius: 5,
                    blurRadius: 7,
                    offset: Offset(0, 3), // changes position of shadow
                  ),
                ]
            ),
            // margin: EdgeInsets.all(15),
            child: Column(
              children: <Widget>[
                Container(
                  margin: const EdgeInsets.only(top: 20),
                  child: const Text("Show your location",
                      textAlign: TextAlign.center,
                      style: TextStyle(color: MyStyle.textLight, fontSize: 24)),
                ),
                const Text("&",
                    textAlign: TextAlign.center,
                    style: TextStyle(color: MyStyle.textLight, fontSize: 24)),
                const Text("See the near by smart toilet",
                    textAlign: TextAlign.center,
                    style: TextStyle(color: MyStyle.textLight, fontSize: 24)),
                SizedBox(height: 20),
                Row(
                  crossAxisAlignment: CrossAxisAlignment.center,
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Expanded(
                        child: Center(
                        child: SizedBox(
                          width: 150,
                          child:ElevatedButton(
                            onPressed: () {
                              Navigator.of(context).pop();
                            },
                            style: ElevatedButton.styleFrom( // styling the button
                            shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(20)),
                            padding: EdgeInsets.only(top: 10, bottom: 10),
                            backgroundColor: Colors.grey, // Button color
                            foregroundColor: Colors.cyan, // Splash color
                            ),
                            child: const Text("Back",
                              style: TextStyle(
                                fontSize: 24,
                                fontWeight: FontWeight.bold,
                                color: Colors.white
                              ),),
                            ),
                        ))),
                    Expanded(
                        child: Center(
                            child: SizedBox(
                              width: 150,
                              child:ElevatedButton(
                                onPressed: () {
                                  Navigator.of(context)
                                      .pushNamed(FinderView.tag);
                                },
                                style: ElevatedButton.styleFrom( // styling the button
                                  shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(20)),
                                  padding: EdgeInsets.only(top: 10, bottom: 10),
                                  backgroundColor: Colors.white, // Button color
                                  foregroundColor: Colors.cyan, // Splash color
                                ),
                                child: const Text("Continue",
                                  style: TextStyle(
                                      fontSize: 24,
                                      fontWeight: FontWeight.bold,
                                      color: MyStyle.purple
                                  ),),
                              ),
                            ))),
                  ],
                ),
                SizedBox(height: 20),
              ],
            ),
          ),
        ],
      ),
    );
  }

  void onSelectAppbarContextMenu(Choice choice) {
    // Causes the app to rebuild with the new _selectedChoice.
    setState(() {
      switch (choice.title) {
        case AppStrings.Menu_Sign_out:
          {

            break;
          }
      }
    });
  }
}
Splashview.dart
import 'dart:io';
import 'dart:ui';
import 'package:flutter/services.dart';
import 'package:connectivity/connectivity.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';
import 'package:smart_toire/screens/landing_view.dart';
import 'package:smart_toire/utils/app_assets.dart';
import 'package:smart_toire/utils/app_strings.dart';
import 'package:smart_toire/utils/app_style.dart';
import 'package:smart_toire/utils/status_dialog.dart';

class SplashView extends StatefulWidget {
  static String tag = 'splash-page';

  const SplashView({super.key});

  @override
  _SplashViewState createState() => _SplashViewState();
}

class _SplashViewState extends State<SplashView> with WidgetsBindingObserver {
  @override
  void initState() {
    //Show status dialog
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    WidgetsBinding.instance
        .addPostFrameCallback((_) => afterFirstFrameRender(context));
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

// Callback After the first frame render completed
  Future<void> afterFirstFrameRender(BuildContext context) async {
    var connectivityResult = await (Connectivity().checkConnectivity());
    if (connectivityResult == ConnectivityResult.mobile ||
        connectivityResult == ConnectivityResult.wifi) {
      // FirebaseConfig config = await BhSharedPreference().getFirebaseConfig();
      await Firebase.initializeApp();
    } else {
      StatusDialog.displayDialogWithAction(context, AppStrings.Lbl_Empty,
          AppStrings.Err_NoConnection, AppStrings.Btn_RETRY, () {
        afterFirstFrameRender(context);
      });
    }
  }


  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: MyStyle.purple,
      body: Column(
        children: <Widget>[
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.center,
              mainAxisAlignment: MainAxisAlignment.center,
              children: <Widget>[
                 const Text(
                  "SMART TOIRE",
                  style: TextStyle(
                    fontSize: 40,
                    fontWeight: FontWeight.bold,
                    color: MyStyle.textWhite,
                  ),
                ),
                Container(
                  width: 250,
                  alignment: Alignment.centerRight,
                  child: const Text(
                    "Home Comfort On The Go",
                    style: TextStyle(
                      fontSize: 14,
                      fontStyle: FontStyle.italic,
                      fontWeight: FontWeight.normal,
                      color: MyStyle.textWhite,
                    ),
                  ),
                ),
              ],
            ),
          ),
          Container(
            width: MediaQuery.of(context).size.width,
            child: Image.asset(
                AppAssets.ImageBanner,
              fit: BoxFit.fitWidth,
            ),
          ),

          const Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.center,
              mainAxisAlignment: MainAxisAlignment.center,
              children: <Widget>[
                Text(
                  "Toilet Location",
                  style: TextStyle(
                    fontSize: 34,
                    fontStyle: FontStyle.normal,
                    fontWeight: FontWeight.bold,
                    color: MyStyle.textWhite,
                  ),
                ),
                Text(
                  "Where you need",
                  style: TextStyle(
                    fontSize: 20,
                    fontStyle: FontStyle.normal,
                    fontWeight: FontWeight.normal,
                    color: MyStyle.textWhite,
                  ),
                ),
              ],
            ),
          ),
          Expanded(
              child: ElevatedButton(
                  onPressed: () {
                    Navigator.of(context)
                        .pushNamed(LandingView.tag);
                  },
                  style: ElevatedButton.styleFrom( // styling the button
                    shape: CircleBorder(),
                    padding: EdgeInsets.all(20),
                    backgroundColor: Colors.white, // Button color
                    foregroundColor: Colors.cyan, // Splash color
                  ),
                  child: const Icon(Icons.arrow_forward, color: MyStyle.textDark)
              )
          ),
          const Text(
            "BY",
            style: TextStyle(
              fontSize: 14,
              fontStyle: FontStyle.normal,
              fontWeight: FontWeight.normal,
              color: MyStyle.textWhite,
            ),
          ),
          const Text(
            "Government of Kerala",
            style: TextStyle(
              fontSize: 14,
              fontStyle: FontStyle.normal,
              fontWeight: FontWeight.normal,
              color: MyStyle.textWhite,
            ),
          ),
        ],
      ),
    );
  }
}
Build.gradle
buildscript {
    ext.kotlin_version = '1.8.20'//'1.7.10'
    ext.multidex_version = '2.0.1'
    repositories {
        google()
        mavenCentral()
    }

    dependencies {
        classpath 'com.android.tools.build:gradle:7.3.0'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        classpath 'com.google.gms:google-services:4.3.13'
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.buildDir = '../build'
subprojects {
    project.buildDir = "${rootProject.buildDir}/${project.name}"
}
subprojects {
    project.evaluationDependsOn(':app')
}

tasks.register("clean", Delete) {
    delete rootProject.buildDir
}





