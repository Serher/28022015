/* ****************************************************************************
 * AnDenix Lootable Player Corpses                                            *
 * andenix@andenix.com                                                        *
 *                                                                            *
 * Last change: 12/27/03                                                      *
 * ************************************************************************** */

 /*
    Modifyed by Tobur 30/03/2004
 */
//#include "ya"
#include "inc_adutils"
//#include "tob_inc_persist"
#include "sl_s_inc2"
#include "tob_pk"
#include "sl_s_pvp_inc"

const int DROP_LOOT = TRUE;//Should we drop the loot?
const int DROP_STOLLEN_GOODS = TRUE;//Will be overriden by setting DROP_LOOT to TRUE
const int LEAVE_BLOOD = FALSE;//Leaves a blood spot (could be off for some reasons)
const int PERCENT_GOLD_TO_DROP = 100;//0 to disable gold dropping on death
const string CEMETARY_WP_TAG = "AD_WP_CEMETARY";//Where we create Master Corpses
const string FUGUE_WP_TAG = "wpdeath";//Where all Dead Falk go
const string FUGUE_EASY_WP_TAG = "EASY_RESS_WP";//Where newbe Dead Falk go
const string HOUSE_LOVE_AREA_TAG = "house_of_love";//Where all Dead Falk go
const string TEMPLE1_TAG = "TEMLE_1"; //Where Self Resurrected Falk go
const string TEMPLE2_TAG = "TEMLE_2"; //Where Self Resurrected Falk go
const string TEMPLE3_TAG = "TEMLE_3"; //Where Self Resurrected Falk go
const string TEMPLE4_TAG = "TEMLE_4"; //Where Self Resurrected Falk go
const string OUTLAW_TAG = "OUTLAW_WP"; //Where Self Resurrected Outlaw go
const string OUTLAW2_TAG = "OUTLAW2_WP"; //Where Self Resurrected Outlaw go
const string OUTLAW3_TAG = "OUTLAW3_WP"; //Where Self Resurrected Outlaw go
const string OUTLAW4_TAG = "OUTLAW4_WP"; //Where Self Resurrected Outlaw go
const string RES_WP_TAG = "Point_Main_Podsolnuh"; //Where Resurrected Falk go
//const string HOUSE_LOVE_AREA_TAG = "FieldsofDanger";//Where all Dead Falk go


// Max Cost Resurrection
const int COST_RES_WO_BODY = 30000;
const int COST_RAISE = 20000;
const int COST_RES = 50000;
const int COST_TRUERES = 100000;
// Type Resurrection
const int TEMPLE_RAISE_DEAD = 1;
const int TEMPLE_RESURRECTION = 2;
const int TEMPLE_TRUE_RES = 3;
const int TEMPLE_RES_WITHOUT_BODY = 4;
const int SCROLL_RAISE_DEAD = 5;
const int SCROLL_RESURRECTION = 6;
const int USER_SPELL_RAISE_DEAD = 7;
const int USER_SPELL_RESURRECTION = 8;
const int USER_SELF_RESURRECTION = 9;

// Destroy all object link to dead Player
void VanishCorpse(object oPC);

//True if oObject is a valid Cropse Item
int GetIsCorpseItem(object oObject);

//Create an Instance of Death Corpse from the MasterCorpse
object CreateCorpse(location lWhere, object oMasterCorpse);

//Creates MasterCorpse from which other instances will be copyed
// Note: Should be creates only once for each player death
object CreateMasterCorpse(location lWhere, object oSourcePC);

//Clears inventory of the object including equipment and
//All carried things in backpack
void ClearInventory(object oChar);

//Destroys one's inventory including GP
void DestroyInventory(object oChar);

//Destroys the equipment of the target
//All weared items, armor and weapons
void DestroyEquipment(object oChar);

//Safely destroys any visible corpse object
void DestroyCorpse(object oCorpse);

// bS upd 22.11.9 @ v367
//Leaves a Corpse from Just Killed NPC
//object DetachCorpse(object oPC, location lWhere);
void DetachCorpse(object oPC, location lWhere);

// bS upd 22.11.9 @ v367
//Used in Module's OnUnAquire scrips to drop carried corpse on the ground
//and in modele OnClientLeave
void DropCorpse(object oCorpseItem, location lLostWhere, int Destroy = TRUE);


//Used in OnDistubed event of DeathCorpse's container object to safely put
//corpse into one's backpack.
void PickUpCorpse(object oCorpseItem);

//Ressurects a Player oPC
int ResurrectPlayer(object oPC, int TypeRes, object oCleric = OBJECT_INVALID);

// Is Player in House of Pain?
int IsPlayerInHouseOfPain(object oPC);
// Is Player in House of Love?
int IsPlayerInHouseOfLove(object oPC);
// Resurrect Player from spell
int Spell_Player_Resurrect(int iLastSpellID);
// Drop random equipped thing
void DropRandom(object oPlayer, object oLootBag);
// Drop another player corpse (if carry), Destroy death corpse in inventory oPC
void DropAnotherPlayerCorpse(object oPC, location lWhere, int Destroy = TRUE);
// End Death Cutscene
void CancelDeathCutscene(object oPlayer);
//
void RemoveAllEffect(object oPlayer);
// Return link on Master Corpse
object GetMasterCorpse(object oPC);
// Destroy player corpse in inventory oPC
void DestroyAnotherPlayerCorpse(object oPC);

//------------------------------------------------------------------------------

object CreateMasterCorpse(location lWhere, object oSourcePC)
{
    // determinate location
    vector vMaster = GetPositionFromLocation(lWhere);
    float fFacing = GetFacingFromLocation(lWhere)+IntToFloat(d100(3));
    vMaster.x += cos(fFacing);
    vMaster.y += sin(fFacing);
    location lMaster = Location(GetAreaFromLocation(lWhere), vMaster, fFacing);

    object oMasterCorpse = CopyObject(oSourcePC, lMaster, OBJECT_INVALID,"master_corpse");
    // make master corpse selectable, for determenation purpose
    ApplyEffectToObject(DURATION_TYPE_INSTANT, EffectDeath(FALSE,FALSE),oMasterCorpse);
    AssignCommand(oMasterCorpse, SetIsDestroyable(FALSE, FALSE, TRUE));
    AssignCommand(oMasterCorpse, ChangeToStandardFaction(oMasterCorpse,STANDARD_FACTION_MERCHANT));
    return oMasterCorpse;
}

//------------------------------------------------------------------------------

object CreateCorpse(location lWhere, object oMasterCorpse)
{
 object oCorpse = CopyObject(oMasterCorpse, lWhere);
 ApplyEffectToObject(DURATION_TYPE_INSTANT, EffectDamage(GetMaxHitPoints(oCorpse)), oCorpse);
 AssignCommand(oCorpse, ChangeToStandardFaction(oCorpse,STANDARD_FACTION_MERCHANT));
 AssignCommand(oCorpse, SetIsDestroyable(FALSE, FALSE, FALSE));
 return oCorpse;
}

//------------------------------------------------------------------------------

void DestroyInventory(object oChar)
{                                                                              //DBM("inc_ad_corpses","DestroyInventory","BEFORE CRASH ");
 int nAmtGold = GetGold(oChar); //Get any gold from the dead creature
 if(nAmtGold) TakeGoldFromCreature(nAmtGold, oChar, TRUE);
 //Get the remaining loot from the dead creature Destroy it
 object oLootEQ = GetFirstItemInInventory(oChar);
 while(GetIsObjectValid(oLootEQ))
 {
   DestroyObject(oLootEQ);
   oLootEQ = GetNextItemInInventory(oChar);
 }
}

//------------------------------------------------------------------------------

void DestroyEquipment(object oChar)
{
 object oItem;

 int i;
 for (i=0;i<=NUM_INVENTORY_SLOTS;++i)
 {
   oItem = GetItemInSlot(i, oChar);
   if(GetIsObjectValid(oItem))
   {
        slWriteLostLog(oItem, "inc_ad_corpses " + "DestroyEquipment");
        DestroyObject(oItem);
   }
 }
}

//------------------------------------------------------------------------------

void ClearInventory(object oChar)
{
 DestroyInventory(oChar);
 DestroyEquipment(oChar);
}

//------------------------------------------------------------------------------

void DestroyCorpse(object oCorpse)
{
  AssignCommand(oCorpse, ClearInventory(oCorpse));
  AssignCommand(oCorpse, SetIsDestroyable(TRUE));
  AssignCommand(oCorpse, DestroyObject(oCorpse));
}

//------------------------------------------------------------------------------
void DestroyLootBag(object oLootBag)
{
    int nAmtGold = GetGold(oLootBag);
    if(nAmtGold)
        TakeGoldFromCreature(nAmtGold, oLootBag, TRUE);
    object oLootEQ = GetFirstItemInInventory(oLootBag);
    while(GetIsObjectValid(oLootEQ))
    {
        DestroyObject(oLootEQ);
        oLootEQ = GetNextItemInInventory(oLootBag);
    }
    AssignCommand(oLootBag, SetIsDestroyable(TRUE));
    AssignCommand(oLootBag, DestroyObject(oLootBag));
}
//------------------------------------------------------------------------------
void CopyObject_(object oItem,location lLootBagLoc,object oLootBag)
{
    CopyObject(oItem, lLootBagLoc, oLootBag);
}
//------------------------------------------------------------------------------
//object DetachCorpse(object oPC, location lWhere)
void DetachCorpse(object oPC, location lWhere)
{
    object oMCorpse = GetMasterCorpse(oPC);
    // destroy dead things, if exist
    if (GetIsObjectValid(oMCorpse))
        VanishCorpse(oMCorpse);

    // Drop Random Thing bF turned off no easy ress anymore
    //if (!GetEasyRess(oPC))
    //ACHTUNG (Ok) Here was drop random before!!! DropRandom(oPC); moved down

    // Create master corpse in graveyard (for last use after drop/pick up & store all var)
    oMCorpse = CreateMasterCorpse(GetLocation(GetWaypointByTag(CEMETARY_WP_TAG)), oPC);
    // destroy items in master corpse (for perfomance reason?)
    DestroyInventory(oMCorpse);

    // create corpse player in place his/her death
    object oCorpse = CreateCorpse(lWhere, oMCorpse);
    // location, where we place corpse
    location lCorpseLoc = GetLocation(oCorpse);

    //bS 22.11.9
    /*bS 29.7.12 Tileset fixed long ago
    object oArea = GetAreaFromLocation(lCorpseLoc);
    if(GetTilesetResRef(oArea)=="tec01")
    {
        vector vLoc = GetPositionFromLocation(lCorpseLoc);
               vLoc = Vector(vLoc.x, vLoc.y, vLoc.z+1.0f);
        lCorpseLoc = Location(oArea, vLoc, 0.0f);
    }*///Eo bS

    // create invisible useable placeable, where stored player token-corpse
    object oCorpseCont = CreateObject(OBJECT_TYPE_PLACEABLE, "ad_corpse_cont", lCorpseLoc);
    AssignCommand(oCorpseCont, SetIsDestroyable(FALSE, FALSE, TRUE));
    object oCorpseItem = CreateItemOnObject(GetGender(oCorpse)?"femalecorpse":"malecorpse", oCorpseCont);
    // create heartbeat object for tracking player token location
    object oCorpseHB = CreateObject(OBJECT_TYPE_PLACEABLE, "ad_corpse_hb", /*bS 22.11.9 lWhere*/lCorpseLoc);
    AssignCommand(oCorpseHB, SetIsDestroyable(FALSE, FALSE, TRUE));

    // create links

    // Master Corpse holds all var
    SetLocalString(oMCorpse,"M_KEY",ID_PC_Key(oPC)); // link to player ID
    SetLocalObject(oMCorpse,"M_PLAYER",oPC); // link to player
    SetLocalString(oMCorpse,"M_NAME",GetDeity(oPC)); // link to player name
    SetLocalObject(oMCorpse,"M_VIS_CORPSE",oCorpse); // link to visible corpse
    SetLocalObject(oMCorpse,"M_INVIS_CORPSE",oCorpseCont); // link to invisible corpse (contaner)
    SetLocalObject(oMCorpse,"M_TOKEN",oCorpseItem); // link to corpse item
    SetLocalObject(oMCorpse,"M_HB_TOKEN",oCorpseHB); // link to heartbeat corpse item
    SetLocalLocation(oMCorpse,"M_TOKEN_LOC",lCorpseLoc); // link to corpse item location
    // links to master corpse
    SetLocalObject(GetModule(),ID_PC_Key(oPC)+"CORPSE_MASTER",oMCorpse); // player link
    SetLocalObject(oCorpse,"CORPSE_MASTER",oMCorpse); // visible corpse link
    SetLocalObject(oCorpseCont,"CORPSE_MASTER",oMCorpse); // invisible corpse link
    SetLocalObject(oCorpseItem,"CORPSE_MASTER",oMCorpse); // corpse item link
    SetLocalObject(oCorpseHB,"CORPSE_MASTER",oMCorpse); // corpse item HB link

    // create loot

    // determinate location
    float fLootBagFacing = GetFacingFromLocation(lCorpseLoc);
    vector vLootBagFacing = GetPositionFromLocation(lCorpseLoc);
    vLootBagFacing.x += sin(fLootBagFacing)*0.5;
    vLootBagFacing.y += cos(fLootBagFacing)*0.5;
    location lLootBagLoc = Location(GetAreaFromLocation(lCorpseLoc),vLootBagFacing,fLootBagFacing);

    object oLootBag = CreateObject(OBJECT_TYPE_PLACEABLE, "pc_lootbag", lLootBagLoc);
    //ACHTUNG (Ok) DROP RANDOM WAS MOVED HERE
    SetLocalInt(oPC, "ToBeSaved", 1);
    AssignCommand(oLootBag, DropRandom(oPC, oLootBag));
    object oItem;
    // Do not fire save char too often on unaqu items

    if (GetIsBPackNonDrop(oPC) && !GetLocalInt(GetModule(),"PK_SYS")) // NewPvP
    {
        // drop skull
        oItem = GetFirstItemInInventory(oPC);
        while(GetIsObjectValid(oItem))
        {
            if (GetTag(oItem)=="tob_pc_scull")
            {
                AssignCommand(oLootBag,CopyObject_(oItem, lLootBagLoc, oLootBag));
                AssignCommand(oLootBag,DestroyObject(oItem));
            }
            oItem = GetNextItemInInventory(oPC);
        }
    }
    else
    {
        // drop gold
        int nIsGold = FALSE;

        if(PERCENT_GOLD_TO_DROP>0 && PERCENT_GOLD_TO_DROP<=100)
        {
            int nGoldAmount = GetGold(oPC);
            if(nGoldAmount>1)
            {
                int nGoldToDrop = (nGoldAmount*PERCENT_GOLD_TO_DROP)/100;
                if(nGoldToDrop>0)
                {
                    AssignCommand(oLootBag, TakeGoldFromCreature(nGoldToDrop, oPC));
                    nIsGold = TRUE;
                }
            }
        }
        // drop inventory
        if(DROP_LOOT)
        {
            oItem = GetFirstItemInInventory(oPC);
            while(GetIsObjectValid(oItem))
            {
                // used copy object faster?
                // Not drop plot & cursed items
                //Mod bS 13.10.2007
                if( GetLocalInt(oItem,"DROPIT") ||
                  !(GetPlotFlag(oItem) || GetItemCursedFlag(oItem)) )
                //Eo Mod bS
                    if (GetBaseItemType(oItem)==BASE_ITEM_LARGEBOX)
                    {
                        slWriteLostLog(oItem, "inc_ad_corpses " +"334 line");
                        AssignCommand(oLootBag,DestroyObject(oItem));
                    }
                    else
                    {
                        AssignCommand(oLootBag,CopyObject_(oItem, lLootBagLoc, oLootBag));
                        slWriteLostLog(oItem, "inc_ad_corpses " +"340 line");
                        AssignCommand(oLootBag,DestroyObject(oItem));
                    }
                oItem = GetNextItemInInventory(oPC);
            }
        }
        else
        {
            if(DROP_STOLLEN_GOODS) // drop only stolen goods
            {
                object oItem = GetFirstItemInInventory(oPC);
                object oNewItem;
                string sOldRes;
                int nStackSize;
                int nIdent;
                int nItemCharges;

                while(GetIsObjectValid(oItem))
                {
                    if(GetStolenFlag(oItem) && !GetPlotFlag(oItem))
                    {
                        sOldRes = GetResRef(oItem);
                        nStackSize = GetItemStackSize(oItem);
                        nItemCharges = GetItemCharges(oItem);
                        nIdent = GetIdentified(oItem);
                        slWriteLostLog(oItem, "inc_ad_corpses " +"365 line");
                        DestroyObject(oItem);

                        oNewItem = CreateItemOnObject(sOldRes, oLootBag, nStackSize);
                        SetIdentified(oNewItem, nIdent);
                        SetItemCharges(oNewItem, nItemCharges);
                    }

                    oItem = GetNextItemInInventory(oPC);
                }
            }
        }
    }
    SetLocalString(oLootBag,"M_NAME",GetDeity(oPC)); // link to player name
    AssignCommand(GetModule(),DelayCommand(HoursToSeconds(20),DestroyLootBag(oLootBag)));
    AssignCommand(oPC, DeleteLocalInt(oPC, "ToBeSaved"));
    ExecuteScript("sl_s_savechar", oPC);
    // return oCorpseCont;
    //bS 11.10.2007
    /*if(GetTag(GetAreaFromLocation(lWhere))=="")
    {
        s
    }
    */
}

//------------------------------------------------------------------------------

void DropCorpse(object oCorpseItem, location lLostWhere, int Destroy = TRUE)
{
    object oMCorpse = GetLocalObject(oCorpseItem, "CORPSE_MASTER");

    if (Destroy)
        DestroyObject(oCorpseItem);

    if(!GetIsObjectValid(oMCorpse))
    {
        return;
    }

    vector vCorpsePos = GetPositionFromLocation(lLostWhere);

    float fFacing = IntToFloat(d100(3));

    vCorpsePos.x += cos(fFacing)*0.25;
    vCorpsePos.y += sin(fFacing)*0.25;

    location lCorpseLoc = Location(GetAreaFromLocation(lLostWhere), vCorpsePos, fFacing);

    object oCorpse = CreateCorpse(lCorpseLoc, oMCorpse);
    SetLocalObject(oMCorpse,"M_VIS_CORPSE",oCorpse); // link to visible corpse
    SetLocalObject(oCorpse,"CORPSE_MASTER",oMCorpse); // visible corpse link

    //bS 22.11.9
    object oArea = GetAreaFromLocation(lCorpseLoc);
    if(GetTilesetResRef(oArea)=="tec01")
    {
        vector vLoc = GetPositionFromLocation(lCorpseLoc);
               vLoc = Vector(vLoc.x, vLoc.y, vLoc.z+1.0f);
        lCorpseLoc = Location(oArea, vLoc, 0.0f);
    }
    //Eo bS

    object oCorpseCont = CreateObject(OBJECT_TYPE_PLACEABLE, "ad_corpse_cont", lCorpseLoc/*bS 22.11.9 GetLocation(oCorpse)*/);
    AssignCommand(oCorpseCont, SetIsDestroyable(FALSE, FALSE, TRUE));
    SetLocalObject(oMCorpse,"M_INVIS_CORPSE",oCorpseCont); // link to invisible corpse (contaner)
    SetLocalObject(oCorpseCont,"CORPSE_MASTER",oMCorpse); // invisible corpse link

    object oNewCorpseItem = CreateItemOnObject(GetGender(oCorpse)?"femalecorpse":"malecorpse", oCorpseCont);
    SetLocalObject(oMCorpse,"M_TOKEN",oNewCorpseItem); // link to corpse item
    SetLocalObject(oNewCorpseItem,"CORPSE_MASTER",oMCorpse); // corpse item link
    SetLocalLocation(oMCorpse,"M_TOKEN_LOC",lCorpseLoc); // link to corpse item location

}

//------------------------------------------------------------------------------

void PickUpCorpse(object oCorpseItem)
{
    object oMCorpse = GetLocalObject(oCorpseItem, "CORPSE_MASTER");
    object oVisibleCorpse =  GetLocalObject(oMCorpse,"M_VIS_CORPSE");
    object oCorpseCont =  GetLocalObject(oMCorpse,"M_INVIS_CORPSE");

    DestroyCorpse(oVisibleCorpse);
    DestroyCorpse(oCorpseCont);
}

//------------------------------------------------------------------------------

int GetIsCorpseItem(object oObject)
{
    return (GetTag(oObject) == "player_corpse");
}


//------------------------------------------------------------------------------
// bS upd 21.11.9 @ v367
// #include "inc_ad_corpses"
void RessurectEnd(object oPC, object oCleric, int iDamaged, int iLostExp);
void RessurectEnd(object oPC, object oCleric, int iDamaged, int iLostExp)
{
                                                                               //DBM("inc_ad_corpses","RessurectEnd","start with " +IToS(iLostExp));
    // Penalty for PK
    int iKillCount = GetKillCount(oPC);
    //    iLostExp = ( (100+((iKillCount>RED_LINE_KK)?RED_LINE_KK:iKillCount) )*iLostExp)/100;
    if (iKillCount > RED_LINE_KK)
       iLostExp =  ((50+RED_LINE_KK)*iLostExp)/50;
    else
       iLostExp =  ((50+iKillCount)*iLostExp)/50;
    //DBM("inc_ad_corpses","RessurectEnd","start");
    if (!GetKilledByPlayer(oPC))
    {
        // Check for death mark (if player dead)
        //nS 20.7.8 if ((GetDeathMark(oPC)) || (GetLocalInt(oPC, "DeathMark")))
        {
            if (ApplyXPPenalty(oPC, iLostExp/2))
                DoStripItems(oPC);
                // ATENTION Put pale master fix with it
        }
    }
    //S
    else
    {
        //bS 20.7.8 if ((GetDeathMark(oPC)) || (GetLocalInt(oPC, "DeathMark")))
        {
           if (ApplyXPPenalty(oPC, iLostExp/4))
               DoStripItems(oPC);
               // ATENTION Put pale master fix with it
        }
    }
    //
    // Clear death mark
    SetDeathMarkOff(oPC);                                                      //DBM("inc_ad_corpses","RessurectEnd","DeathMarkSettedOff");
    //SetTokenLocation
    //Part By Feron 15.10.11
    string sPC_ID = ID_PC_Key(oPC);
    SetCampaignInt("Siala", sPC_ID + "Clone", 0);
    SetLocalInt(oPC, "DeathMark", FALSE);
    ///////////////////////////////////////////
    //bS 5.9.9 - Fix for ressurecters's logout case
    object oToken = GetLocalObject(oPC, "TokenofPersistence");
    SetLocalInt(oToken, "OweXP", GetLocalInt(oToken, "OweXP")-iLostExp);
    DeleteLocalString(oToken, "RName");
    //
    ApplyEffectToObject(DURATION_TYPE_INSTANT, EffectVisualEffect(VFX_IMP_RESTORATION_LESSER), oPC);
    RemoveAllEffect(oPC);
    ApplyEffectToObject(DURATION_TYPE_INSTANT, EffectHeal(GetMaxHitPoints(oPC)), oPC);
    if (iDamaged)
    {
        DelayCommand(2.0, ApplyEffectToObject(DURATION_TYPE_INSTANT, EffectDamage(GetCurrentHitPoints(oPC)-10), oPC)); //bZ 02.05.11

        /*
            iDamaged = GetMaxHitPoints(oPC);
        if( GetCurrentHitPoints(oPC) < iDamaged)
            iDamaged = GetCurrentHitPoints(oPC);
        ApplyEffectToObject(DURATION_TYPE_INSTANT,EffectDamage(iDamaged-10), oPC);
        */
    }
    // for PKed
    if (GetKilledByPlayer(oPC)) //
    {
        //if (GetEasyRess(oPC))  bF The system was turned off
        //    SetEasyRessOff(oPC);
       //else // áîëåçíü âîñêðåøåíèß äëß ÏÂÏøíèêîâ
        //{
        effect eBad = EffectAttackDecrease(2);
        eBad = EffectLinkEffects(EffectACDecrease(2),eBad);
        eBad = EffectLinkEffects(EffectCurse(2,2,0),eBad);
        eBad = SupernaturalEffect(eBad);
        ApplyEffectToObject(DURATION_TYPE_TEMPORARY, eBad, oPC, RoundsToSeconds(30/*iLostExp/2*/));
        //}
        SetKilledByPlayerOff(oPC);
    }
    SetPVPReputation(oPC);
    SetFactionVisual(oPC);
     // save persistence data
    SavePersistenceData(oPC);
     // save character
    ExecuteScript("sl_s_savechar", oPC);
}
void MoveToLight(object oPC, object oCleric, int iDamaged, int iLostExp, location lResLoc);
void MoveToLight(object oPC, object oCleric, int iDamaged, int iLostExp, location lResLoc)
{
    //DBM("inc_ad_corpses","MoveToLight","start");
    object oArea = GetArea(oPC);
    if (GetIsObjectValid(oArea))
    {
        SetTokenLocation(oPC, lResLoc);
        if (!(GetTag(oArea)==HOUSE_LOVE_AREA_TAG))
        {
            //DBM("inc_ad_corpses","MoveToLight","before RessurectEnd called");
            RessurectEnd(oPC, oCleric, iDamaged, iLostExp);
        }
        else
        {
            AssignCommand(oPC, ClearAllActions(TRUE));
            AssignCommand(oPC, JumpToLocation(lResLoc));
            DelayCommand(2.0f,MoveToLight(oPC, oCleric, iDamaged, iLostExp, lResLoc));
        }
    }
    else
        DelayCommand(2.0f,MoveToLight(oPC, oCleric, iDamaged, iLostExp, lResLoc));
}



int GetXPPenaltyCoef(int iExp) // bZ 12.01.14
{
    int nCoef;
    if(iExp<153000)
        nCoef = 2;
    else if (iExp<190000)
        nCoef = 3;
    else if (iExp<300000)
        nCoef = 4;
    else if (iExp<406000)
        nCoef = 5;
    else if (iExp<496000)
        nCoef = 6;
    else if (iExp<595000)
        nCoef = 7;
    else if (iExp<666000)
        nCoef = 8;
    else if (iExp<741000)
        nCoef = 9;
    else
        nCoef = 10;

return nCoef;
}

int ResurrectPlayer(object oPC, int TypeRes, object oCleric = OBJECT_INVALID)
{
    if(!GetIsObjectValid(oPC))
        return 0;                                                              //DBM("inc_ad_corpses","ResurrectPlayer","start");

    object oMCorpse = GetMasterCorpse(oPC);
    string sNamePC = GetDeity(oPC);
    int iLostExp;
    int iExp = GetXP(oPC);
    int nCoef = GetXPPenaltyCoef(iExp); // Ïîòåðÿ îïûòà èçìåíåíà Àñôàëüòîì 20.09.2013. Ñäåëàíî ëó÷øèì äëÿ ëó÷øèõ... //upd bZ 12.01.14  Óáðàë çà ëó÷øèì :)
    int iClericLevel = GetLevelByClass(CLASS_TYPE_CLERIC,oCleric);

    location lResLoc = GetLocation(GetObjectByTag(RES_WP_TAG));
    if (GetIsObjectValid(oCleric))
    {
        object oArea = GetArea(oCleric);
        int nAreaFlag = GetLocalInt(oArea,"slAreaFlags");                            //DBM("inc_ad_corpses","CheckForPermaAggression","Area Flag" + IToS(nAreaFlag));

        int nAF = nAreaFlag & 4;/*00100 - mask */                                  //DBM("inc_ad_corpses","CheckForPermaAggression","After Mask" + IToS(nAreaFlag));
        if( nAF )
        {                                                                          //DBM("inc_ad_corpses","CheckForPermaAggression","Rak area");
            if(GetIdFaction(oPC) == 1)     // Valiostrec
            {                                                                        //DBM("inc_ad_corpses","CheckForPermaAggression","Vallca resajut");
                if(!GetIdFaction(oCleric) != 1)    {
                   //DBM("sl_s_pvp_inc","CheckForPermaAggression","Vallec resaet!");
                    FloatingTextStringOnCreature("Âîñêðåøàòü âðàãà íà òåððèòîðèè ñâîåãî ãîñóäàðñòâà - ïðåñòóïëåíèå!", OBJECT_SELF);
                    return -1;
                }
            }
        }
        else
        { //Val
            nAF = nAreaFlag & 2; /*00010 - mask */                                 //DBM("inc_ad_corpses","CheckForPermaAggression","After Second Mask" + IToS(nAreaFlag));
            if ( nAF )
            {                                                                  //DBM("inc_ad_corpses","CheckForPermaAggression","Val area");
                if(GetIdFaction(oPC) == 2)     // rac
                {                                                                  //DBM("inc_ad_corpses","CheckForPermaAggression","Raka resajut");
                   //DBM("inc_ad_corpses","CheckForPermaAggression","Resatels Fraction!" + IToS(GetIdFaction(oCleric)));
                    if(GetIdFaction(oCleric) != 2)
                    {                                                              //DBM("inc_ad_corpses","CheckForPermaAggression","Rak resaet!");
                        FloatingTextStringOnCreature("Âîñêðåøàòü âðàãà íà òåððèòîðèè ñâîåãî ãîñóäàðñòâà - ïðåñòóïëåíèå!", OBJECT_SELF);
                        return -1;
                    }
                }
           }
        }

        vector vResLoc = GetPosition(oCleric);
        float fResLoc = GetFacing(oCleric);
//        fResLoc = (fResLoc>360.0f)?fResLoc-360.0f:fResLoc ;
        vResLoc.x = vResLoc.x + cos(fResLoc);
        vResLoc.y = vResLoc.y + sin(fResLoc);
        lResLoc = Location(GetArea(oCleric),vResLoc,fResLoc);
    }

    int iDamaged = FALSE ;                                                     //DBM("inc_ad_corpses","ResurrectPlayer","TypeRes " + IToS(TypeRes));
    switch (TypeRes)
    {
        case TEMPLE_RAISE_DEAD:
            AssignCommand(oCleric,ActionCastFakeSpellAtLocation(SPELL_RAISE_DEAD,lResLoc));
            iDamaged = TRUE ;
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-45;
            //else
                iLostExp=iExp/40/nCoef;
            break;
        case TEMPLE_RESURRECTION:
            AssignCommand(oCleric,ActionCastFakeSpellAtLocation(SPELL_RESURRECTION,lResLoc));
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-60;
            //else
                iLostExp=iExp/50/nCoef;
            break;
        case TEMPLE_TRUE_RES:
            AssignCommand(oCleric,ActionCastFakeSpellAtLocation(SPELL_RESURRECTION,lResLoc));
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-75;
            //else
                iLostExp=iExp/60/nCoef;
            break;
        case TEMPLE_RES_WITHOUT_BODY:
            AssignCommand(oCleric,ActionCastFakeSpellAtLocation(SPELL_RAISE_DEAD,lResLoc));
            iDamaged = TRUE ;
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-37;
            //else
                iLostExp=iExp/37/nCoef;
            break;
        case SCROLL_RAISE_DEAD:
            AssignCommand(oCleric,ActionCastFakeSpellAtLocation(SPELL_RAISE_DEAD,lResLoc));
            iDamaged = TRUE ;
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-37;
            //else
                iLostExp=iExp/37/nCoef;
            break;
        case SCROLL_RESURRECTION:
            AssignCommand(oCleric,ActionCastFakeSpellAtLocation(SPELL_RESURRECTION,lResLoc));
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-52;
            //else
                iLostExp=iExp/52/nCoef;
            break;
        case USER_SPELL_RAISE_DEAD:
            iDamaged = TRUE ;
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-(37+(iClericLevel*3/2-12));
            //else
                iLostExp=iExp/(37+(iClericLevel*3/2-12))/nCoef;
            break;
        case USER_SPELL_RESURRECTION:
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-(52+(iClericLevel*3/2-18));
            //else
                iLostExp=iExp/(52+(iClericLevel*3/2-18))/nCoef;
            break;
        case USER_SELF_RESURRECTION:
            //if (GetKilledByPlayer(oPC)) // óáèòûå èãðîêàìè îïûò íå òåðßþò
            //    iLostExp = 100-30;
            //else
                iLostExp=iExp/30;
            lResLoc = GetLocalLocation(oPC,"Resurrection_Loc");
            DeleteLocalLocation(oPC,"Resurrection_Loc");
            break;
        default:
            return 0;
            break;
    }                                                                           //DBM("inc_ad_corpses","VanishCorpse","before");
    // destroy all death things
    VanishCorpse(oMCorpse);                                                     //DBM("inc_ad_corpses","VanishCorpse","after");

    // NewPvP for when you were ressurected on the enemy terrotiry
    if(!GetLocalInt(GetModule(),"PK_SYS"))
    {
        int nAreaFlag = GetLocalInt(GetAreaFromLocation(lResLoc),"slAreaFlags");
        int nAF=nAreaFlag&4;//00100 - maska
        if(nAF!=0)  //òåðèòîðèÿ èíñàííû
        {
            if(GetIdFaction(oPC)==1)
                SetPermanentAggressor(oPC,"2");
        }
        else
        {
            nAF=nAreaFlag&2;
            if(nAF!=0)  //òåðèòîðèÿ âàëèîñòðà
            {
                if(GetIdFaction(oPC)==2)
                    SetPermanentAggressor(oPC,"1");
            }
        }
    }
    //

    //Move player to Res Place
    //FadeToBlack(oPC,FADE_SPEED_FASTEST);
    AssignCommand(oPC, JumpToLocation(lResLoc));                                //DBM("inc_ad_corpses","VanishCorpse","after jump");
    //bS 5.9.9 - Fix for ressurecters's logout case
    object oToken = GetLocalObject(oPC, "TokenofPersistence");
    SetLocalInt(oToken, "OweXP", GetLocalInt(oToken, "OweXP")+iLostExp);
    SetLocalString(oToken, "RName", GetDeity(oCleric));                         //DBM("inc_ad_corpses","ResurrectPlayer","before MoveToLight called");
    DelayCommand(2.0f,MoveToLight(oPC, oCleric, iDamaged, iLostExp, lResLoc));  //DBM("inc_ad_corpses","VanishCorpse","after move");
    return TRUE;
}

//------------------------------------------------------------------------------

void VanishCorpse(object oMCorpse)
{
    string sNamePC = GetLocalString(oMCorpse,"M_NAME");
    object oCorpseItem = GetLocalObject(oMCorpse,"M_TOKEN");
    object oCorpseCont = GetLocalObject(oMCorpse,"M_INVIS_CORPSE");
    object oCorpse = GetLocalObject(oMCorpse,"M_VIS_CORPSE");
    object oCorpseHB = GetLocalObject(oMCorpse,"M_HB_TOKEN");
    object oPC = GetLocalObject(oMCorpse,"M_PLAYER");
    object oPossessor = GetItemPossessor(oCorpseItem); // get possessor corpse item

    if (GetIsPC(oPossessor))
    {
        string sFeedBack;
        sFeedBack = "Òåëî "+sNamePC+" âíåçàïíî èñ÷åçëî";
        SendMessageToPC(oPossessor, sFeedBack);
    }

    DestroyObject(oCorpseItem);//Destroy a Death Corpse item
    DestroyCorpse(oCorpse);//Destroys Visible Corpse of the player
    DestroyCorpse(oCorpseHB);//Destroys HB Corpse
    DestroyCorpse(oCorpseCont);//Destroys a container
    DeleteLocalObject(GetModule(),GetLocalString(oMCorpse,"M_KEY")+"CORPSE_MASTER");
    DeleteLocalObject(oMCorpse,"M_KEY"); // link to player ID
    DestroyCorpse(oMCorpse);//Destroys Master Corpse from "Limbo"

}


//------------------------------------------------------------------------------
int IsPlayerInHouseOfPain(object oPC)
{
    string  sAreaTag = GetTag(GetArea(oPC));
    return  sAreaTag=="porog_area" || sAreaTag=="l01" /*17.10.9|| sAreaTag=="l02" || sAreaTag=="l03" || sAreaTag=="I04" || sAreaTag=="l05" */|| sAreaTag==HOUSE_LOVE_AREA_TAG;
}
//------------------------------------------------------------------------------
int IsPlayerInHouseOfLove(object oPC)
{
    return GetTag(GetArea(oPC))==HOUSE_LOVE_AREA_TAG;
}


// No Magic Area
int IsRestrictedArea(object oPC)
{
    /*bS 09.03.2007
    string sAreaTag = GetTag(GetArea(oPC));
    return (sAreaTag=="AveFortDungeon") || (sAreaTag=="AveFortPainRoom") || (sAreaTag=="AveFortJail") || (sAreaTag=="PenalServituMin")|| (sAreaTag=="PenalServitude") || (sAreaTag=="PenalBarack");
    */
    return GetLocalInt(GetArea(oPC), "NOM");
    //Eo bS
}
//------------------------------------------------------------------------------
void CreateScroll(int iTypeScroll, object Owner)
{
    switch (iTypeScroll)
    {
        case SPELL_RAISE_DEAD:
            CreateItemOnObject("nw_it_spdvscr501",Owner);
            break;
        case SPELL_RESURRECTION:
            CreateItemOnObject("nw_it_spdvscr702",Owner);
            break;
    }
}
int Spell_Player_Resurrect(int iLastSpellID)
{
    object oItemSpellCast = GetSpellCastItem();
    object oTarget = GetSpellTargetObject();
    int iItemCast = GetIsObjectValid(oItemSpellCast);
    object oCleric = OBJECT_SELF;
    int iScrollCast = ( (iItemCast) && (GetBaseItemType(oItemSpellCast)==BASE_ITEM_SPELLSCROLL) );

    if (GetTag(oTarget)=="player_corpse")
    {
        SendMessageToPC(oCleric,"Ïåðåä ïðîâåäåíèåì îáðÿäà âîñêðåøåíèÿ, ïîëîæè òðóï íà çåìëþ");
        if (iScrollCast)
            DelayCommand(1.0f,CreateScroll(iLastSpellID,oCleric));
        return TRUE;
    }
    if (GetTag(oTarget)!="ad_corpse_cont")
    {
        return FALSE;
    }
    else
    {
        object oMasterCorpse = GetLocalObject(oTarget,"CORPSE_MASTER"); // get master corpse
        string sOwner = GetLocalString(oMasterCorpse,"M_NAME");
        object oResurected = GetLocalObject(oMasterCorpse,"M_PLAYER");
        if (!GetIsObjectValid(oResurected))
        {
            SendMessageToPC(oCleric,sOwner+" ïîêèíóë ýòîò ìèð, ïîýòîìó âîñêðåøåíèå íåâîçìîæíî");
            if (iScrollCast)
                DelayCommand(1.0f,CreateScroll(iLastSpellID,oCleric));
            return TRUE;
        }
        if (!IsPlayerInHouseOfLove(oResurected))
        {
            SendMessageToPC(oCleric,"Äóøà "+sOwner+" íå äîñòèãëà Äîìà Ëþáâè, ïîýòîìó âîñêðåøåíèå íåâîçìîæíî");
            if (iScrollCast)
                DelayCommand(1.0f,CreateScroll(iLastSpellID,oCleric));
            return TRUE;
        }
        if (GetLocalInt(GetArea(oCleric),"NoRaise"))
        {
            SendMessageToPC(oCleric,"Äóøà óìåðøåãî íå ìîæåò áûòü âîçâðàùåíà ñþäà. Âàì íóæíî íàéòè áîëüåå ñïîêîéíîå ìåñòî.");
            if (iScrollCast)
                DelayCommand(1.0f,CreateScroll(iLastSpellID,oCleric));
            return TRUE;
        }

        int iResult = FALSE;
        switch (iLastSpellID)
        {
            case SPELL_RAISE_DEAD:
                if (iItemCast)
                    iResult = ResurrectPlayer(oResurected,SCROLL_RAISE_DEAD,oCleric);
                else
                    iResult = ResurrectPlayer(oResurected,USER_SPELL_RAISE_DEAD,oCleric);
                return iResult;
                break;

            case SPELL_RESURRECTION:
                if (iItemCast)
                    iResult = ResurrectPlayer(oResurected,SCROLL_RESURRECTION,oCleric);
                else
                    iResult = ResurrectPlayer(oResurected,USER_SPELL_RESURRECTION,oCleric);
                return iResult;
                break;

            default:
                return FALSE;
                break;

        }

    }

    return TRUE;
}

//------------------------------------------------------------------------------
void DropRandom(object oPlayer, object oLootBag)
{             //DBM("inc_ad_corpses","DropRandom","start");
    if(GetIsEquipNonDrop(oPlayer)&& !GetLocalInt(GetModule(),"PK_SYS"))
    {
        //DBM("inc_ad_corpses","DropRandom","RETURN"+ OToS(oPlayer)+" NoDropSetted");
        return;
    }
    int iFate = d100();
    if(GetTag(GetArea(oPlayer))=="StartLoc")
    {
        string sMessage = "[DropRandom] "+GetDeity(oPlayer)+": No Drop Equipped on StartLoc";
        AssignCommand(oPlayer, SpeakString(sMessage, TALKVOLUME_SILENT_SHOUT));
        WriteTimestampedLogEntry(sMessage);
        return;
    }
    int nDropDecrease = GetLocalInt(oPlayer,"RD");
    if( nDropDecrease>2) nDropDecrease*=3;
    else                 nDropDecrease*=4;
    //bS 20.6.12 AssignCommand(oPlayer, SpeakString(GetDeity(oPlayer)+": iFate: "+IntToString(iFate)+" ?> "+IntToString(80+nDropDecrease), TALKVOLUME_SILENT_SHOUT));
    //ACHTUNG (CHECK) return this statement
    if (iFate > 92/*bS 01.10.2007 -GetKillCount(oPlayer)+*/+nDropDecrease) //92 was 80 //bZ 08.07.2014
    {
        int iSlot = d10();
        if(iSlot==1) iSlot=0;//bS 01.10.2007 - Do not drop armor - drop helmet instead
        object oItem = GetItemInSlot(iSlot, oPlayer);                          //DBM("inc_ad_corpses","DropRandom","oItem" + OToS(oItem));
        if (!GetIsObjectValid(oItem))   //bF This is not a fix but a joke
            oItem = GetItemInSlot(((iSlot=Random(11))==1)?2:iSlot,oPlayer);
        if ( (GetIsObjectValid(oItem)) && (GetLocalInt(oItem,"DROPIT") || (!GetItemCursedFlag(oItem) && !GetPlotFlag(oItem))) )//bS 20.7.2012
        {          //DBM("inc_ad_corpses","DropRandom","check ok" );
            //bS 4.8.8 uncommented old mech temp
            //ACHTUNG (CHECK) What is the reason for copy item?      I commented this undo this in case of troubles
            //CopyItem(oItem, oPlayer, TRUE);
            AssignCommand(oPlayer, SpeakString(GetDeity(oPlayer)+" drops "+GetName(oItem), TALKVOLUME_SILENT_SHOUT));
            SendMessageToPC(oPlayer,"Òåáå íå ïîâåçëî: â ìîìåíò ñìåðòè òû ïîòåðÿë "+GetName(oItem));
            slRecreateItem(oItem, oLootBag);
            if (GetTag(oItem)=="lw_flag")
                SendMessageToPC(oPlayer,"Òû ïîòåðÿë Ñâÿùåííóþ Õîðóãâü! Âîèíû Ñâåòà íå ïðîñòÿò òåáå ýòîãî!");
            //It seems that there is no need for this lines do not uncomment
            //slWriteLostLog(oItem, "DROP RANDOM");
            //DestroyObject(oItem);
            //bS 4.8.8 slRecreateItem(oItem, oPlayer);
        }
    }
    if (iFate > 98) // critical failure  //was 95
        DropRandom(oPlayer, oLootBag);
}

//------------------------------------------------------------------------------
void DropAnotherPlayerCorpse(object oPC, location lWhere, int Destroy = TRUE)
{
    // drop corpse another player
    // if  player has another player corpse?
    if (GetIsObjectValid(GetItemPossessedBy(oPC, "player_corpse")))
    {
        // drop all player corpse
        object oInvItem = GetFirstItemInInventory(oPC);
        while (GetIsObjectValid(oInvItem))
        {
            if (GetTag(oInvItem)=="player_corpse")
                DropCorpse(oInvItem, lWhere, Destroy);
            oInvItem = GetNextItemInInventory(oPC);
        }
    }
}
//------------------------------------------------------------------------------
void CancelDeathCutscene(object oPlayer)
{
    //SetCommandable(TRUE,oPlayer);
    object oMod = GetModule();
    FadeFromBlack(oPlayer,FADE_SPEED_SLOWEST);
    effect eEffect = EffectVisualEffect(VFX_IMP_RESTORATION_GREATER);
    ApplyEffectToObject(DURATION_TYPE_INSTANT, eEffect, oPlayer);
    SetCameraFacing(GetFacing(oPlayer));
    SetCutsceneMode(oPlayer,FALSE);
    SetLocalInt(oMod,ID_PC_Key(oPlayer)+"NotEndDeath",FALSE);
    RemoveAllEffect(oPlayer);
}
//------------------------------------------------------------------------------
void RemoveAllEffect(object oPlayer)
{
    effect eBad = GetFirstEffect(oPlayer);
    while(GetIsEffectValid(eBad))
    {
        RemoveEffect(oPlayer, eBad);
        eBad = GetNextEffect(oPlayer);
    }
}
//------------------------------------------------------------------------------
object GetMasterCorpse(object oPC)
{
    object oMCorpse = GetLocalObject(GetModule(),ID_PC_Key(oPC)+"CORPSE_MASTER");
    if (GetIsObjectValid(oMCorpse))
        return oMCorpse;
    else
        return OBJECT_INVALID;
}
//------------------------------------------------------------------------------
void DestroyAnotherPlayerCorpse(object oPC)
{
    // destroy all player corpse items, if exist
    if (GetIsObjectValid(GetItemPossessedBy(oPC, "player_corpse")))
    {
        object oInvItem = GetFirstItemInInventory(oPC);
        while (GetIsObjectValid(oInvItem))
        {
            if (GetTag(oInvItem)=="player_corpse")
                DestroyObject(oInvItem);
            oInvItem = GetNextItemInInventory(oPC);
        }
    }
}
//------------------------------------------------------------------------------
void GarantyJumpToPorog(object oPlayer)
{
    if ((GetDeathMark(oPlayer))||(GetLocalInt(oPlayer, "DeathMark"))) //if Player marked for death
    {
        if (!IsPlayerInHouseOfPain(oPlayer)) // if he/she not in death location
        {
            //Just in case
            AssignCommand(oPlayer, ClearAllActions(TRUE));
            //Move player to Fugue
            AssignCommand(oPlayer, JumpToObject(GetWaypointByTag(FUGUE_WP_TAG)));
            DelayCommand(15.0f,GarantyJumpToPorog(oPlayer));
        }
    }
}
//void main() {}

