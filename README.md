```swift

//
//  ContentView.swift
//  SleepAnalysisSampleApp
//
//  Created by paige shin on 2022/10/18.
//

/// https://benoitpasquier.com/sleep-healthkit/
/// https://www.appsloveworld.com/swift/100/89/apple-healthkit-rem-deep-light-sleep-analysis
/// https://www.wwdcnotes.com/notes/wwdc22/10005/
/// https://developer.apple.com/videos/play/wwdc2022/10005

import SwiftUI
import HealthKit

struct ContentView: View {
    
    let healthStore = HKHealthStore()
    @State var startDate = Date()
    @State var timeText: String = ""
    @State var started: Bool = false
    @State var timer = Timer.publish(every: 0.01, on: .main, in: .common).autoconnect()
    @State var results: [String] = []
    
    var body: some View {
        VStack {
            
            if results.count > 0 {
                List {
                    ForEach(results, id: \.self) { text in
                        Text(text)
                    }
                }
            }
       
            Text(self.timeText)
                .padding(.bottom)
            
            Button("Start") {
                self.timer = Timer.publish(every: 0.01, on: .main, in: .common).autoconnect()
                self.started = true
                self.startDate = Date()
            }
            .font(.system(size: 24))
            .padding(.bottom)
            
            Button("Finish") {
                self.finish()
            }
            .font(.system(size: 24))
        }
        .padding()
        .onAppear {
            self.askPermission()
        }
        .onReceive(self.timer) { timer in
            if self.started {
                self.updateTime()
            }
        }
    }
    
    
    
    func updateTime() {
        let currentTime = Date.timeIntervalSinceReferenceDate
        
        //Find the difference between current time and start time.
        var elapsedTime: TimeInterval = currentTime - startDate.timeIntervalSinceReferenceDate
        
        // print(elapsedTime)
        //  print(Int(elapsedTime))
        
        //calculate the minutes in elapsed time.
        let minutes = UInt8(elapsedTime / 60.0)
        elapsedTime -= (TimeInterval(minutes) * 60)
        
        //calculate the seconds in elapsed time.
        let seconds = UInt8(elapsedTime)
        elapsedTime -= TimeInterval(seconds)
        
        //find out the fraction of milliseconds to be displayed.
        let fraction = UInt8(elapsedTime * 100)
        
        //add the leading zero for minutes, seconds and millseconds and store them as string constants
        
        let strMinutes = String(format: "%02d", minutes)
        let strSeconds = String(format: "%02d", seconds)
        let strFraction = String(format: "%02d", fraction)
        
        //concatenate minuets, seconds and milliseconds as assign it to the UILabel
        self.timeText = "\(strMinutes):\(strSeconds):\(strFraction)"
    }
    
    func finish() {
        self.saveSleepAnalysis()
        self.retrieveSleepAnalysis()
        self.timer.upstream.connect().cancel()
    }
    
    func askPermission() {
        let typestoRead = Set([
            HKObjectType.categoryType(forIdentifier: HKCategoryTypeIdentifier.sleepAnalysis)!
        ])
        
        let typestoShare = Set([
            HKObjectType.categoryType(forIdentifier: HKCategoryTypeIdentifier.sleepAnalysis)!
        ])
        
        self.healthStore.requestAuthorization(toShare: typestoShare, read: typestoRead) { (success, error) -> Void in
            if success == false {
                NSLog(" Display not allowed")
            }
        }
    }
    
    func saveSleepAnalysis() {
        
        // alarmTime and endTime are NSDate objects
        if let sleepType = HKObjectType.categoryType(forIdentifier: HKCategoryTypeIdentifier.sleepAnalysis) {
            
            
            if #available(iOS 16.0, *) {
                for sleepValue in HKCategoryValueSleepAnalysis.allAsleepValues {
                    
                    print("Sleep Value: \(sleepValue)")
                    
                    switch sleepValue {
                    case HKCategoryValueSleepAnalysis.asleepCore:
                        print("Sleep Core")
                    case HKCategoryValueSleepAnalysis.inBed:
                        print("In Bed")
                    case HKCategoryValueSleepAnalysis.asleepDeep:
                        print("Asleep Deep")
                    case HKCategoryValueSleepAnalysis.asleepREM:
                        print("REM")
                    case HKCategoryValueSleepAnalysis.awake:
                        print("Awake")
                    case HKCategoryValueSleepAnalysis.asleepUnspecified:
                        print("Asleep Unspecified")
                    @unknown default:
                        print("None")
                    }
                    
                    
                    // we create our new object we want to push in Health app
                    let object = HKCategorySample(type:sleepType, value: sleepValue.rawValue, start: self.startDate, end: Date())
                    
                    // at the end, we save it
                    healthStore.save(object, withCompletion: { (success, error) -> Void in
                        
                        if error != nil {
                            // something happened
                            return
                        }
                        
                        if success {
                            print("My new data was saved in HealthKit")
                            
                        } else {
                            // something happened again
                        }
                        
                    })
                }
                
                
            } else {
                
                let allAsleepValues: Set<HKCategoryValueSleepAnalysis> = [
                    HKCategoryValueSleepAnalysis.inBed,
                    HKCategoryValueSleepAnalysis.awake,
                    HKCategoryValueSleepAnalysis.asleep,
                ]
                
                for sleepValue in allAsleepValues {
                    // we create our new object we want to push in Health app
                    let object = HKCategorySample(type:sleepType, value: sleepValue.rawValue, start: self.startDate, end: Date())
                    
                    // at the end, we save it
                    healthStore.save(object, withCompletion: { (success, error) -> Void in
                        
                        if error != nil {
                            // something happened
                            return
                        }
                        
                        if success {
                            print("My new data was saved in HealthKit")
                            
                        } else {
                            // something happened again
                        }
                        
                    })
                }
                
     
                
            }
            
        }
        
    }
    
    
    func retrieveSleepAnalysis() {
        
        // first, we define the object type we want
        if let sleepType = HKObjectType.categoryType(forIdentifier: HKCategoryTypeIdentifier.sleepAnalysis) {
            
            // Use a sortDescriptor to get the recent data first
            let sortDescriptor = NSSortDescriptor(key: HKSampleSortIdentifierEndDate, ascending: false)
            
            // we create our query with a block completion to execute
            let query = HKSampleQuery(sampleType: sleepType, predicate: nil, limit: 30, sortDescriptors: [sortDescriptor]) { (query, tmpResult, error) -> Void in
                
                if error != nil {
                    
                    // something happened
                    return
                    
                }
                
                if let result = tmpResult {
                    
                    // do something with my data
                    for item in result {
                        if let sample = item as? HKCategorySample {
                            let value = (sample.value == HKCategoryValueSleepAnalysis.inBed.rawValue) ? "InBed" : "Asleep"
                            print("Healthkit sleep: \(sample.startDate) \(sample.endDate) - value: \(value)")
                            if #available(iOS 16.0, *) {
                                switch sample.value {
                                case HKCategoryValueSleepAnalysis.asleepCore.rawValue:
                                    self.results.append("Sleep Core, \(sample.startDate)")
                                case HKCategoryValueSleepAnalysis.inBed.rawValue:
                                    self.results.append("In Bed, \(sample.startDate)")
                                case HKCategoryValueSleepAnalysis.asleepDeep.rawValue:
                                    self.results.append("Asleep Deep, \(sample.startDate)")
                                case HKCategoryValueSleepAnalysis.asleepREM.rawValue:
                                    self.results.append("REM, \(sample.startDate)")
                                case HKCategoryValueSleepAnalysis.awake.rawValue:
                                    self.results.append("Awake, \(sample.startDate)")
                                case HKCategoryValueSleepAnalysis.asleepUnspecified.rawValue:
                                    self.results.append("Asleep Unspecified, \(sample.startDate)")
                                default: print("None")
                                }
                            } else {
                                switch sample.value {
                                case HKCategoryValueSleepAnalysis.inBed.rawValue:
                                    self.results.append("In Bed, \(sample.startDate)")
                                case HKCategoryValueSleepAnalysis.awake.rawValue:
                                    self.results.append("Awake, \(sample.startDate)")
                                case HKCategoryValueSleepAnalysis.asleep.rawValue:
                                    self.results.append("Asleep, \(sample.startDate)")
                                default: print("None")
                                }
                            }
                        }
                    }
                }
            }
            
            // finally, we execute our query
            healthStore.execute(query)
        }
    }
    
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}


```
