void setup(){
  pinMode(5, OUTPUT);    // mentioning the GPIO Number
}

void loop(){             // To run repeatedly
  digitalWrite(5, HIGH); 
  delay(1000);
  digitalWrite(5, LOW);  // Fixed: added pin number and changed to LOW
  delay(1000);
}
