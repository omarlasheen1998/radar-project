# Sensor Fusion Nanodegree ,Udacity, Project of Radar Target Generation and Detection

## Project Layout:
<img src="./ProjectRequirement.png" width="700" />


---
#### 1. Radar Specifications 
* Frequency of operation = 77GHz
* Max Range = 200m
* Range Resolution = 1 m
* Max Velocity = 100 m/s

#### 2. User Defined Range and Velocity of target

* define the target's initial position and velocity. 
* Note : Velocity remains contant
```
r0=150; % Target Initial Range
v0= 50; % Target Velocity
```

#### 3. FMCW Waveform Generation
* Design the FMCW waveform by giving the specs of each of its parameters.
* Calculate the Bandwidth (B), Chirp Time (Tchirp) and Slope (slope) of the FMCW chirp using the requirements above.

* Operating carrier frequency of Radar 
```
fc= 77e9; %Operating carrier frequency of Radar             
c=3e8;  %speed of electromagnetic wave
Range_resolution=1;
Max_velocity= 100;
Max_range = 200;
B=c/(2*Range_resolution);
Tchirp = 5.5*2*Max_range/c;
slope=B/Tchirp;
fprintf('Bsweep = %.2f \t Tchirp = %f \t slope = %.2f\n', B, Tchirp, slope); 

```                                                          
* The number of chirps in one sequence. 
* Its ideal to have `2^value` for the ease of running the FFT for Doppler Estimation. 
```
Nd=128;                   % # of doppler cells OR # of sent periods % number of chirps
```
* The number of samples on each chirp. 
```
Nr=1024;                  % for length of time OR # of range cells
```
* Timestamp for running the displacement scenario for every sample on each chirp
```
t=linspace(0,Nd*Tchirp,Nr*Nd); %total time for samples
```
* Creating the vectors for Tx, Rx and Mix based on the total samples input.
```
Tx=zeros(1,length(t)); %transmitted signal
Rx=zeros(1,length(t)); %received signal
Mix = zeros(1,length(t)); %beat signal
```
* Similar vectors for range_covered and time delay.
```
r_t=zeros(1,length(t)); % range_covered
td=zeros(1,length(t)); % time delay
```

#### 4. Signal generation and Moving Target simulation
```
for i=1:length(t)         
    
    
    %For each time stamp update the Range of the Target for constant velocity. 
    
    range = r0 + t(i)*v0;
    
    %For each time sample we need update the transmitted and
    %received signal. 
     td    = 2*range / c;
    tnew  = t(i)-td;
    Tx(i) = cos( 2*pi*( fc*t(i) + (0.5*slope*t(i)^2) ) );
    Rx(i) = cos( 2*pi*( fc*tnew + (0.5*slope*tnew^2) ) );
    
    %Now by mixing the Transmit and Receive generate the beat signal
    %This is done by element wise matrix multiplication of Transmit and
    %Receiver Signal
    Mix(i) = Tx(i).*Rx(i);
    
end
```

#### 5. Range Measurement

* Reshape the vector into Nr*Nd array. 
* Nr and Nd here would also define the size of Range and Doppler FFT respectively.
```
newMix = reshape(Mix,[Nr,Nd]);

%run the FFT on the beat signal along the range bins dimension (Nr) and
%normalize.
fft_signal = fft(newMix);
% Take the absolute value of FFT output
fft_signal = abs(fft_signal/max(max(fft_signal)));
 % *%TODO* :
% Output of FFT is double sided signal, but we are interested in only one side of the spectrum.
% Hence we throw out half of the samples.
fft_signal = fft_signal(1:length(t)/2+1);

 % plot FFT output 
L=length(t);
figure ('Name','Range from First FFT')
f = L*(0:L/2)/L;
plot(f,fft_signal) 
axis ([0 200 0 1]);
xlabel('range Axis')
ylabel('normalized output Axis')
```
* Simulation Result
<img src="./fft_radar.png" width="700" />

#### 6. Range Doppler Response
* The 2D FFT implementation is already provided here. 
* This will run a 2DFFT on the mixed signal (beat signal) output and generate a range doppler map.
* You will implement CFAR on the generated RDM Range Doppler Map Generation.
* The output of the 2D FFT is an image that has reponse in the range and doppler FFT bins. 
* So, it is important to convert the axis from bin sizes to range and doppler based on their Max values.
```
Mix=reshape(Mix,[Nr,Nd]);

% 2D FFT using the FFT size for both dimensions.
sig_fft2 = fft2(Mix,Nr,Nd);

% Taking just one side of signal from Range dimension.
sig_fft2 = sig_fft2(1:Nr/2,1:Nd);
sig_fft2 = fftshift (sig_fft2);
RDM = abs(sig_fft2);
RDM = 10*log10(RDM) ;
%use the surf function to plot the output of 2DFFT and to show axis in both
%dimensions
doppler_axis = linspace(-100,100,Nd);
range_axis = linspace(-200,200,Nr/2)*((Nr/2)/400);
figure,surf(doppler_axis,range_axis,RDM);
xlabel('doppler Axis')
ylabel('range Axis')
zlabel('RDM')
```
* Simulation Result
<img src="./fft2_radar.png" width="700" />

#### 7. CFAR implementation
* Select number of training cells for range and doppler axis
* Select number of gaurd cells for range and doppler axis
* Select the offset by signal to noise ratio in db
* normalize the RDM
* Create a vector to store noise_level for each iteration on training cells

* Design a loop such that For every iteration sum the signal level within all the training
* To sum convert the RDM value from logarithmic to linear using db2pow function and average the summed values for all of the training

* After averaging convert RDM value back to logarithimic using pow2db.
* Further add the offset to it to determine the threshold.
* compare the signal under Cell Under Test with this threshold. If the CUT level > threshold assign it a value of 1, else equate it to 0
* Slide Window through the complete Range Doppler Map
* As the CUT cannot be located at the edges of matrix,Hence,few cells will not be thresholded. To keep the map size same set those values to 0.
* Display the CFAR output using the Surf function
```
Tr = 8;
Td = 10;

%Select the number of Guard Cells in both dimensions around the Cell under 
%test (CUT) for accurate estimation
Gr = 4;
Gd = 4;

% offset the threshold by SNR value in dB
offset = 1.4;

%Create a vector to store noise_level for each iteration on training cells
%noise_level = zeros(1,1);


%design a loop such that it slides the CUT across range doppler map by
%giving margins at the edges for Training and Guard Cells.
%For every iteration sum the signal level within all the training
%cells. To sum convert the value from logarithmic to linear using db2pow
%function. Average the summed values for all of the training
%cells used. After averaging convert it back to logarithimic using pow2db.
%Further add the offset to it to determine the threshold. Next, compare the
%signal under CUT with this threshold. If the CUT level > threshold assign
%it a value of 1, else equate it to 0.


   % Use RDM[x,y] as the matrix from the output of 2D FFT for implementing
   % CFAR
   
RDM = RDM/max(max(RDM));

for i = Tr+Gr+1:(Nr/2)-(Gr+Tr)
    for j = Td+Gd+1:Nd-(Gd+Td)
        
       %Create a vector to store noise_level for each iteration on training cells
        noise_level = zeros(1,1);
        
        % Calculate noise SUM in the area around CUT
        for p = i-(Tr+Gr) : i+(Tr+Gr)
            for q = j-(Td+Gd) : j+(Td+Gd)
                if (abs(i-p) > Gr || abs(j-q) > Gd)
                    noise_level = noise_level + db2pow(RDM(p,q));
                end
            end
        end
        
        % Calculate threshould from noise average then add the offset
        threshold = pow2db(noise_level/(2*(Td+Gd+1)*2*(Tr+Gr+1)-(Gr*Gd)-1));
        threshold = threshold + offset;
        CUT = RDM(i,j);
        
        if (CUT < threshold)
            RDM(i,j) = 0;
        else
            RDM(i,j) = 1;
        end
        
    end
end

% The process above will generate a thresholded block, which is smaller 
%than the Range Doppler Map as the CUT cannot be located at the edges of
%matrix. Hence,few cells will not be thresholded. To keep the map size same
% set those values to 0. 
RDM(union(1:(Tr+Gr),end-(Tr+Gr-1):end),:) = 0;  % Rows
RDM(:,union(1:(Td+Gd),end-(Td+Gd-1):end)) = 0;  % Columns 

%display the CFAR output using the Surf function like we did for Range
%Doppler Response output.
%figure,surf(doppler_axis,range_axis,'replace this with output');
figure('Name','CA-CFAR Filtered RDM')
surf(doppler_axis,range_axis,RDM);
xlabel('doppler Axis')
ylabel('range Axis')
zlabel('normalized RDM')
colorbar;
```
* Simulation Result
<img src="./cfar_output.png" width="700" />
