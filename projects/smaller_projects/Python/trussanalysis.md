---
layout: default
title: Portfolio |  Warren-Truss Structural Analysis
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Truss Analysis
This program performs truss analysis somewhat accurately, with only a few minor issues near the joints.


```python
import math
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import re 
import pprint

"""
[HEADER]: utility functions
"""
#return a dictionary with all the values where the key contains a string
def FilterString(dic, filterString):
    return {k: v for k, v in dic.items() if filterString in k}
def NumberFromString(str):
    num = re.findall('\d+',str)
    num = [int(i) for i in num][0]
    return num
#filter dictionary
def FilterDictionary(dic, removeKeys):
    newDic = {}
    for key, val in dic.items():
        for key, val in dic.items():
            if not any(str in key for str in removeKeys):
                newDic[key] = val
    return newDic

#takes value (x,y)
def CalculateMagnitude(vector):
    dist = math.sqrt((vector[0]**2) + (vector[1]**2))
    return dist
#takes two vectors (x,y)
def CalculateDotProduct(vector1,vector2):
    return (vector1[0]*vector2[0]) + (vector1[1]*vector2[1])

def CalculateDirectionVector(vertexPositions):
    directionVector = (vertexPositions[1][0] - vertexPositions[0][0], vertexPositions[1][1] - vertexPositions[0][1])
    return directionVector

#takes vector (x,y)
def CalculateNormalVector(vector):
    mag = CalculateMagnitude(vector)
    return [vector[0]/mag,vector[1]/mag]

def AngleBetweenVectors(vector1,vector2):
    dotProd = CalculateDotProduct(vector1,vector2)
    vect1Mag = CalculateMagnitude(vector1)
    vect2Mag = CalculateMagnitude(vector2)
    magMult = vect1Mag*vect2Mag
    cal = dotProd / (vect1Mag*vect2Mag)
    angle = abs(math.acos(cal))
    return angle

"""
[HEADER]: construct shape functions
"""
def BuildVertexes(segmentCount,width,height):
    vertexDict = {}
    for i in range(0,segmentCount):
        vertexDict[f"B{i}"] = (width*i,0)
        vertexDict[f"T{i}"] = (width*i,height)
    return vertexDict
"""
input: dictionary of vertexes labelled with T for top and B for bottom, with an integer for index: T1
return: dictionary of members, with the coordinates of both vertexes. the key is combined. it starts with the bottom
        horizontal connections are done in reverse: B2B1
        diagonal connections alternate: B1T2, B2T2
"""
def ConnectVertexes(vertexDict):
    filterString = "B"
    
    trussMembers = {}
    #get the length of the bottom vertexes
    dictLength = len(FilterString(vertexDict,filterString))
    if dictLength < 1:
        return None
    trussMembers["B0T0"] = (vertexDict["B0"],vertexDict["T0"])
    for i in range(1,dictLength):
        i_previous = i-1
        
        #connect vertically
        start = f"B{i}"
        end = f"T{i}"
        trussMembers[start+end] = (vertexDict[start],vertexDict[end])
        
        #connect horizontally
        start = f"B{i_previous}"
        end = f"B{i}"
        trussMembers[start+end] = (vertexDict[start],vertexDict[end])
        
        start = f"T{i_previous}"
        end = f"T{i}"
        trussMembers[start+end] = (vertexDict[start],vertexDict[end])
        
        if i % 2 == 0:
            start = f"B{i}"
            end = f"T{i_previous}"
            trussMembers[start+end] = (vertexDict[start],vertexDict[end])
        else:
            start = f"B{i_previous}"
            end = f"T{i}"
            trussMembers[start+end] = (vertexDict[start],vertexDict[end])
    return trussMembers

"""
this code will work by having a compound dictionary (dictionary in a dictionary)
the dictionary is in this format:
        key (vertexKey): {key(connectedTrussKey): value(force magnitude of member, direction is stored in the connected truss dictionary)}
the dictionary is used to take all forces acting on a joint, and transfer shared calculations to other joints
each element in the dictionary represents the joints, where the value is all the forces and the member that caused it
"""
def CreateForceCompoundDictionaries(vertexDict,connectedTruss):
    forceDict = {}
    for key, _ in vertexDict.items():
        forceDict[key] = {}
        splitDict = FilterString(dic=connectedTruss,filterString=key)
        for truss_key,_ in splitDict.items():
            forceDict[key][truss_key] = None
    return forceDict
def CreateMemberForceDictionary(memberDictionary):
    newDic = {}
    for key,_ in memberDictionary.items():
        newDic[key] = None
    return newDic

def AddLoad(vertexKey, forceMagnitude, forceDict):
    forceDict[vertexKey]["LOAD"] = forceMagnitude
def AddPinSupport(vertexKey, forceDict):
    forceDict[vertexKey]["PIN"] = [None,None] #two values for x and y
def AddRollerSupport(vertexKey, forceDict):
    forceDict[vertexKey]["ROLL"] = None

def IsDeterminate(trussMemberDict,trussJointDict, reactionForcesCount):
    if int(2 * len(trussJointDict)) == int(len(trussMemberDict) + reactionForcesCount):
        return True
    else:
        return False

def PlotTruss(trussDictionary,vertexDictionary,memberForceDictionary,filterKeys,pinKey,rollKey, loadKey,hasMemberLabels=True,hasForceLabels=True,hasJointLables=True,colorMembers=False,hasLegend = True):
    plt.figure()
    for key, value in trussDictionary.items():
        x_values = [value[0][0],value[1][0]]
        y_values = [value[0][1],value[1][1]]
        
        labelXCoordinate = (x_values[0] + x_values[1]) / 2
        labelYCoordinate = (y_values[0] + y_values[1]) / 2
        hasFilt = False
        for str in filterKeys:
            if str in key:
                hasFilt = True
                
        if hasFilt:
            plt.plot(x_values,y_values,color='lightgrey')
        else:
            if hasMemberLabels:
                plt.text(labelXCoordinate,labelYCoordinate,key,fontsize=4,ha='center',color='blue')
            if(memberForceDictionary[key]is not None):
                if(memberForceDictionary[key] > 0):
                    col = "red"
                else:
                    col = "darkorange"
                if hasForceLabels:
                    plt.text(labelXCoordinate,labelYCoordinate+0.01,f"{memberForceDictionary[key]:.2f}",fontsize=6,ha='center',color=col)
            if colorMembers:
                plt.plot(x_values,y_values,color=col)
            else:
                plt.plot(x_values,y_values,color='black')
    for key, value in vertexDictionary.items():
        x_value, y_value = value[0], value[1]
        labelYCoordinates = (y_value - 0.02) if "B" in key else (y_value + 0.01)
        if hasJointLables:
            plt.plot(x_value,y_value,color='blue', marker='.')
            plt.text(x_value,labelYCoordinates,key,fontsize=8,ha='center',color='blue')
    ax = plt.gca()
    ax.set_aspect('equal', adjustable='box')
    #plt.plot( [trussDictionary["B1B2"][0][0],trussDictionary["B1B2"][0][1]], [trussDictionary["B1B2"][1][0],trussDictionary["B1B2"][1][1]])
    plt.title("Method of Joints Analysis")
    plt.xlabel("horizontal")
    plt.ylabel("vertical")
    
    #dummy points to expand plot area
    plt.plot(vertexDictionary["T0"][0],vertexDictionary["T0"][1]*2,color='white')
    plt.plot(vertexDictionary["B0"][0],-vertexDictionary["T0"][1],color='white')
    
    #plot load and support positions
    plt.plot(vertexDictionary[pinKey][0],vertexDictionary[pinKey][1]-0.003,color='green', marker='^',markersize=10)
    plt.plot(vertexDictionary[rollKey][0],vertexDictionary[rollKey][1]-0.003,color='limegreen', marker='^',markersize=10)
    plt.plot(vertexDictionary[loadKey][0],vertexDictionary[loadKey][1]-0.003,color='purple', marker='v',markersize=10)
    
    #plt.grid(True)
    if hasLegend:
        plt.legend(loc='upper left',fontsize='8',handles=[
            mpatches.Patch(color='red', label='Tension'),
            mpatches.Patch(color='darkorange', label='Compression'),
            mpatches.Patch(color='blue', label='joints'),
            mpatches.Patch(color='green', label='pin support'),
            mpatches.Patch(color='limegreen', label='roller suppport'),
            mpatches.Patch(color='Purple', label='load point')
        ],handlelength=2,handletextpad=2,ncol=3)
    plt.tight_layout()
    plt.show()

"""
[HEADER]: calculation functions
"""
#function to fill zeros into zero truss members through definition, allowing calculation in joints with a lot of unknowns such as where two diagonals converge
def CalculateZeros(forceDictionary,memberDictionary,memberForceDictionary):
    PushValue(forceDictionary,"B1T1",0)
    PushMemberValue(memberForceDictionary,"B1T1",0)
    for _,members in forceDictionary.items():
        hasSpecial = False
        for key,_ in members.items():
            if ("LOAD" in key) or ("PIN" in key) or ("ROLL" in key):
                hasSpecial = True
                break
        if len(members) == 2 and not hasSpecial:
            for key,_ in members.items():
                PushValue(forceDictionary,key,0)
                PushMemberValue(memberForceDictionary,key,0)
        if len(members) == 3 and not hasSpecial:
            keys = list(members.keys())
            for i in range(len(keys)):
                for j in range(i+1, len(keys)):
                    vector1 = CalculateDirectionVector(memberDictionary[keys[i]])
                    vector2 = CalculateDirectionVector(memberDictionary[keys[j]])
                    angle = AngleBetweenVectors(vector1, vector2)
                    if angle == 0 or angle == 180 or angle == 360:  # vectors are in the same direction
                        # find the other key and set its value to 0
                        other_key = keys[3 - i - j]
                        members[other_key] = (0, 0)
                        PushValue(forceDictionary,other_key,0)
                        PushMemberValue(memberForceDictionary,other_key,0)
        
#negative values imply compression, position are tension
#calculate the force along the two members across the load. the code does not support loads that are not at the end of the truss
def LoadCalculations(forceDictionary,trussDictionary,memberForceDictionary):
    totalCalcAngle = None #since all diagonal angles are the same, it only needs to be calculated once at the first instance of a diagonal line
    for key, value in forceDictionary.items():
        if isinstance(value, dict) and value.get("LOAD"):
            loadForce = value['LOAD']
            horizontalMembers = {}
            verticalMembers = {}
            diagonalMembers = {}
            for ky,val in value.items():
                if ky == 'LOAD':
                    continue
                nums = re.findall('\d+',ky)
                nums = [int(i) for i in nums]
                if ('B' in ky or 'T' in ky) and not ('B' in ky and 'T' in ky):
                    horizontalMembers[ky] = val
                    continue
                if nums[0] != nums[1]:
                    diagonalMembers[ky] = val
                    continue
                if 'B' in ky and 'T' in ky:
                    verticalMembers[ky] = val
                    continue
            netXForce = 0
            if len(diagonalMembers) != 0:
                for dkey, dvalue in diagonalMembers.items():
                    dirVector = CalculateDirectionVector(trussDictionary[dkey]) #turns (x,y)(x,y) into (x,y)
                    normalVector = CalculateNormalVector(dirVector)
                    if totalCalcAngle is None:
                        """
                        calculate the angle between the normalvector of the member against the y-axis. 
                        x-axis is not used since it's direction yields different results for each member
                        the angle is subtracted from 90 degrees to offset the 90 degree shift by using the y-axis over the x-axis
                        """
                        totalCalcAngle = math.radians(90) - AngleBetweenVectors(normalVector,(0,1))
                    diagonalMemberForce = -(loadForce / math.sin(totalCalcAngle))
                    xvalue = -(diagonalMemberForce * math.cos(totalCalcAngle))
                    netXForce += xvalue
                    PushValue(forceDictionary,dkey,diagonalMemberForce)
                    PushMemberValue(memberForceDictionary,dkey,diagonalMemberForce)
            if len(verticalMembers) !=0:
                pass
            if len(horizontalMembers) != 0:
                for hkey, _ in horizontalMembers.items():
                    xvalue = (netXForce)
                    PushValue(forceDictionary,hkey,xvalue)
                    PushMemberValue(memberForceDictionary,hkey,xvalue)
                    break
    return totalCalcAngle
#calculate the moment, and thus the forces along the joints allowing the completion of the calculations
def MomentReactionForces(forceDictionary,trussDictionary,memberForceDictionary,pinjointKey, rollerjointKey,loadjointKey,loadForce,width,diagonalAngles):
    pinNum = NumberFromString(pinjointKey)
    rollerNum = NumberFromString(rollerjointKey)
    loadNum = NumberFromString(loadjointKey)
    
    loadDistanceFromPin = abs(loadNum - pinNum) * width
    rollerDistanceFromPin = abs(rollerNum - pinNum) * width
    rollerReaction = -((loadDistanceFromPin * loadForce) / rollerDistanceFromPin)
    
    pinVerticalReaction = -(rollerReaction + loadForce)
    pinHorizontalReaction= None #this is calculated at the end, since the forces along the bottom members will always be the same, so setting it via the diagonal reaction gives unbalanced forces, despite having no applied x force
    forceDictionary[pinjointKey]["PIN"] = [pinHorizontalReaction,pinVerticalReaction]
    forceDictionary[rollerjointKey]["ROLL"] = [rollerReaction]

    pinForce = forceDictionary[pinjointKey]['PIN'][1]
    horizontalMembers = {}
    verticalMembers = {}
    diagonalMembers = {}
    for ky,val in forceDictionary[pinjointKey].items():
        if ky == 'PIN':
            continue
        nums = re.findall('\d+',ky)
        nums = [int(i) for i in nums]
        if nums[0] != nums[1] and "T" in ky and "B" in ky:
            diagonalMembers[ky] = val
            continue
    if len(diagonalMembers) != 0:
        for dkey, _ in diagonalMembers.items():
            dirVector = CalculateDirectionVector(trussDictionary[dkey]) #turns (x,y)(x,y) into (x,y)
            normalVector = CalculateNormalVector(dirVector)
            
            totalCalcAngle = math.radians(90) - AngleBetweenVectors(normalVector,(0,1))
            diagonalMemberForce = -(pinForce / math.sin(diagonalAngles)) if math.sin(diagonalAngles) != 0 else 0
            PushValue(forceDictionary,dkey,diagonalMemberForce)
            PushMemberValue(memberForceDictionary,dkey,diagonalMemberForce)
    return pinForce, rollerReaction
                
def ContinueCalculations(forceDictionary,memberForceDictionary,diagonalAngles,loopCounter=0):
    loopCounter += 1
    for key, value in reversed(forceDictionary.items()):
        isTop = False
        if "T" in key:
            isTop = True
        horizontalMembers = {}
        verticalMembers = {}
        diagonalMembers = {}
        netX = 0
        netY = 0
        NoneValues = 0
        for ky,val in value.items():
            if val == None:
                NoneValues += 1
            if ky == 'LOAD':
                continue
            if ky =='ROLL':
                netY += val[0]
                continue
            if ky =='PIN':
                netY += val[1]
                continue
            nums = re.findall('\d+',ky)
            nums = [int(i) for i in nums]
            if ('B' in ky or 'T' in ky) and not ('B' in ky and 'T' in ky):
                horizontalMembers[ky] = val
                continue
            if nums[0] != nums[1]:
                diagonalMembers[ky] = val
                continue
            if 'B' in ky and 'T' in ky:
                verticalMembers[ky] = val
                continue
        if NoneValues > 2:
            continue
        
        if len(verticalMembers) > 0:
            for ky, val in verticalMembers.items():
                if val is not None:
                    netY += val
        if len(diagonalMembers) > 0:
            noneVals = []
            for ky, val in diagonalMembers.items():
                dirNums = re.findall('\d+',ky)
                dirNums = [int(i) for i in nums]
                isForward = True #forward refers to bottom left to top right such as key B0T1, when false refers to keys such as B3T2
                if dirNums[0] > dirNums[1]:
                    isForward = False
                if val is not None:
                    yVal = val * math.sin(diagonalAngles)
                    xVal = val * math.cos(diagonalAngles)
                    
                    #logic determining the direction of the x and y based on whether it slants forward or not, or whether it is on top
                    #this logic is required since the direction of the forces for example compression, are inverted on the top and further inverted based on which way it is pointing
                    #logic derived through several diagrams using positive and negative signs to represent direction (+up and right, -down and left)
                    if isForward and isTop:
                        xVal = -xVal
                        yVal = -yVal
                    elif not isForward:
                        xVal = -xVal if not isTop else xVal
                        yVal = -yVal
                        
                    netY += yVal
                    netX += xVal
                else:
                    noneVals.append(ky)
            if len(noneVals) == 1:
                memberForce = (netY / math.sin(diagonalAngles))
                PushValue(forceDictionary,noneVals[0],memberForce)
                PushMemberValue(memberForceDictionary,noneVals[0],memberForce)
                    
        if len(horizontalMembers) > 0:
            noneVals = []
            for ky, val in horizontalMembers.items():
                dirNums = re.findall('\d+',ky)
                dirNums = [int(i) for i in nums]
                isForward = True #forward refers to bottom left to top right such as key B0T1, when false refers to keys such as B3T2
                if dirNums[0] > dirNums[1]:
                    isForward = False
                if val is not None:
                    netX = netX + val if isForward else netX - val
                    continue
                else:
                    noneVals.append(ky)
            if len(noneVals) == 1:
                memberForce = netX
                loopCounter = 0
                PushValue(forceDictionary,noneVals[0],memberForce)
                PushMemberValue(memberForceDictionary,noneVals[0],memberForce)
    
    #loop to control the recursive calls. a flag is used so that the function is complete before starting the next
    hasNone = False
    for _, val in memberForceDictionary.items():
        if(val == None):
            hasNone = True
            break
    if hasNone:
        if loopCounter > 50:
            return None
        ContinueCalculations(forceDictionary,memberForceDictionary,diagonalAngles,loopCounter)
                
                
def PushValue(forceDictionary,memberKey,pushValue):
    for _, dic in forceDictionary.items():
        for key, _ in dic.items():
            if key == memberKey:
                dic[key] = pushValue
                
def PushMemberValue(memberForceDictionary,memberKey,pushValue):
    memberForceDictionary[memberKey] = pushValue

def AnalyseSafety(forceMemberDictionary,memberDictionary,elasticModulus,memberWidth):
    areaMomentInertia = (memberWidth ** 4) / 12
    bucklingForceTop = (math.pi**2)*(elasticModulus)*(areaMomentInertia)
    for key, _ in forceMemberDictionary.items():
        memberMagnitude = (CalculateMagnitude(CalculateDirectionVector(memberDictionary[key])))
        bucklingForce = bucklingForceTop / (memberMagnitude**2)
        if forceMemberDictionary[key] > bucklingForce:
            print(f"Truss Failed at structure {key} with force {forceMemberDictionary[key]}")
            return False
    print("The truss is safe")
    return True


#main function to call the other functions. Do not change this
def Main():
    vertexDict = BuildVertexes(segmentCount=bayNumber,width=trussWidth,height=trussHeight)
    connectedTruss = ConnectVertexes(vertexDict=vertexDict)
    
    #filter the connected truss from values not needed
    filteredConnectedTruss = FilterDictionary(connectedTruss,ignoreKeys)
    filteredVertexDict = FilterDictionary(vertexDict,ignoreKeys)
    
    forceCompoundDictionary = CreateForceCompoundDictionaries(filteredVertexDict,filteredConnectedTruss)
    forceMemberDictionary = CreateMemberForceDictionary(filteredConnectedTruss)
    
    isDeterminate = IsDeterminate(trussMemberDict=filteredConnectedTruss,trussJointDict=forceCompoundDictionary,reactionForcesCount=3)
    
    AddPinSupport(forceDict=forceCompoundDictionary,vertexKey=pinsupportKey)
    AddRollerSupport(forceDict=forceCompoundDictionary,vertexKey=rollerSupportKey)
    
    
    if(isDeterminate):
        AddLoad(vertexKey=loadKey,forceMagnitude = force,forceDict=forceCompoundDictionary)    
        angle = LoadCalculations(forceCompoundDictionary,filteredConnectedTruss,forceMemberDictionary)
        pin,roll = MomentReactionForces(forceCompoundDictionary,trussDictionary=filteredConnectedTruss,memberForceDictionary=forceMemberDictionary,
                             pinjointKey=pinsupportKey,rollerjointKey=rollerSupportKey,loadjointKey=loadKey,loadForce=force,width=trussWidth,diagonalAngles=angle)
        CalculateZeros(forceCompoundDictionary,filteredConnectedTruss,forceMemberDictionary)
        ContinueCalculations(forceDictionary=forceCompoundDictionary,memberForceDictionary=forceMemberDictionary,diagonalAngles=angle)
        
        trussSafety = AnalyseSafety(forceMemberDictionary,filteredConnectedTruss,elasticModulus,memberWidth)
        print("force dictionary")
        print(forceCompoundDictionary)
        print("member dictionary")
        print(filteredConnectedTruss)
        
        pp = pprint.PrettyPrinter(indent=4) 
        outputResults = {
            "Truss is safe":trussSafety,
            "Pin Support Reaction": f"{pin}N",
            "Roller Support Reaction:": f"{roll}N",
            "Load": f"{force}N",
            "Is Determinate": isDeterminate,
            "Member Count": f"{len(filteredConnectedTruss)}",
            "Truss Span Length": f"{(bayNumber-1) * trussWidth}m",
            "Member Dimensions": f"width={memberWidth}m, length={CalculateMagnitude(CalculateDirectionVector(filteredConnectedTruss['B0T1'])):.3}m(diagonal),{trussWidth}m (double if vertical beams are ignored)"
        }
        pp.pprint(outputResults)
        
        PlotTruss(hasMemberLabels=diagramMemberLabels,hasJointLables=diagramJointLabels,hasForceLabels=diagramForceLabels,colorMembers=diagramColorMembers,
                  hasLegend = diagramLegend,trussDictionary=connectedTruss,vertexDictionary=vertexDict,
                  memberForceDictionary=forceMemberDictionary,filterKeys=ignoreKeys,
                  pinKey=pinsupportKey,rollKey=rollerSupportKey,loadKey=loadKey)

#use values in SI units
if __name__ == "__main__":
    #adjust these variables
    bayNumber = 12
    bayNumber += 1
    
    trussWidth = 0.04 #4 centimeters
    trussHeight = 0.07 #7 centimeters
    
    ignoreKeys = ["T0",f"T{bayNumber-1}"] #don't change this
    
    loadKey = f"B{bayNumber-1}"
    pinsupportKey = "B0"
    rollerSupportKey = "B2"
    force = -5 #load force for a weight of 0.5Kg: f = mg = 0.5*10 N
    

    elasticModulus = 3 * 10**9 #the 10**9 is to convert Gpa to pa
    memberWidth = 0.004 #4 mm
    
    #plot settings
    diagramMemberLabels = False
    diagramJointLabels = True
    diagramForceLabels = True
    diagramColorMembers = False
    diagramLegend = True
    
    Main()
```